# Pulse Hardware I/O Domain Specification

This document defines the Hardware I/O domain of the Pulse project protocol.  
It covers discovery, configuration and monitoring of audio hardware devices:

- device enumeration,
- input/output channel topology,
- sample rate and buffer size,
- clock and latency reporting,
- hot-plug events and failures.

Pulse coordinates hardware I/O configuration between Aura and Signal:

- **Signal** talks to the operating system / driver stack (CoreAudio, ASIO, ALSA, WASAPI, JACK, etc.).
- **Pulse** exposes a stable, platform-agnostic view of devices and stream
  parameters to Aura, and mediates configuration changes.
- **Aura** presents device options to the user and issues configuration
  commands.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Hardware I/O Concepts](#2-hardware-io-concepts)  
  - [2.1 Devices and Streams](#21-devices-and-streams)  
  - [2.2 Channels and Routing](#22-channels-and-routing)  
  - [2.3 Sample Rate and Buffer Size](#23-sample-rate-and-buffer-size)  
  - [2.4 Clock Source and Latency](#24-clock-source-and-latency)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Device Discovery and Selection](#31-device-discovery-and-selection)  
  - [3.2 Stream Configuration](#32-stream-configuration)  
  - [3.3 Channel Mapping](#33-channel-mapping)  
  - [3.4 Refresh and Diagnostics](#34-refresh-and-diagnostics)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Device and Stream Events](#41-device-and-stream-events)  
  - [4.2 Channel Mapping Events](#42-channel-mapping-events)  
  - [4.3 Error and Health Events](#43-error-and-health-events)  
- [5. Snapshot and Persistence Semantics](#5-snapshot-and-persistence-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

The Hardware I/O domain abstracts host audio devices and their configuration so
that:

- Aura can present a unified UI for device selection and stream settings,
- Pulse can maintain a stable configuration model,
- Signal can manage platform-specific audio API details.

Key responsibilities:

- enumerate input and output devices,
- select active devices for the engine,
- configure sample rate and buffer size (subject to driver capabilities),
- define channel ordering and usage (e.g. which Channels feed which outputs),
- report latency, clock source, and hardware health.

Hardware configuration is largely **machine-specific**; project files may store
preferences, but cannot assume the same hardware exists on other machines.

**Note:** This domain covers audio I/O devices (interfaces, sound cards). Control surfaces, MIDI controllers, and assistive hardware are handled through separate IPC mechanisms. Pulse maintains mapping rules for control surfaces and sends Feedback Intents to Signal for hardware output (LEDs, displays), but the low-level device enumeration and I/O transport is handled by Signal's hardware domain.

---

## 2. Hardware I/O Concepts

### 2.1 Devices and Streams

A **hardware device** is a host-level audio interface:

- built-in audio,
- external USB/Thunderbolt/PCIe interfaces,
- virtual devices (e.g. JACK, loopback drivers).

Each device exposes one or more **streams** (often logically combined), with:

- input channels,
- output channels,
- supported sample rates,
- supported buffer sizes.

Pulse presents each device with:

- `deviceId` (stable within a session; derived from OS identifiers),
- `name` and `vendor`,
- `isDefaultInput`, `isDefaultOutput` flags,
- driver/API type (for diagnostics).

The engine may support:

- a single duplex device (common case),
- or separate input and output devices (where the driver/API permits it).

### 2.2 Channels and Routing

Each device exposes **channels**:

- input channels (hardware → Signal),
- output channels (Signal → hardware).

Channels have:

- `channelIndex` (zero-based within input or output),
- `name` (label reported by the OS/driver where available),
- `grouping` (e.g. “Input 1–2”, “Monitor L/R”).

Logical routing from internal Channels to hardware I/O is handled by the
Routing and Channel domains; the Hardware I/O domain focuses on exposing and
selecting the physical endpoints.

#### 2.2.1 Logical Roles

Hardware I/O supports **logical roles** that express project-level intent for
how audio should be routed to physical devices. Roles are stable across
machines and represent user intent rather than specific hardware.

Standard roles include:

- `main` (main output, typically stereo),
- `phones` (headphone output),
- `cue1`, `cue2` (cue outputs),
- `controlRoom` (control room monitoring),
- `talkbackIn` (talkback input),
- and other domain-specific roles.

User-defined roles (e.g., `user.*`) are also supported, allowing projects to
define custom routing intentions.

Roles express *project-level intent* and should remain stable across machines.
A project that specifies `role.main` should work on any machine, even if the
actual device differs.

#### 2.2.2 Device Aliases

**Device aliases** are human-friendly identifiers for physical devices and their
channel groups. Aliases provide a machine-level realisation of roles, allowing
the same project to work across different hardware configurations.

Examples of device aliases:

- `interface.studioMain` (a studio interface),
- `interface.laptop` (built-in laptop audio),
- `interface.liveRig` (a live performance interface).

Aliases may also reference specific channel groups within a device:

- `interface.studioMain:main` (main output pair on the studio interface),
- `interface.studioMain:phones` (headphone outputs on the studio interface).

This allows fine-grained control over which physical channels serve which
logical purpose, even within a single device.

#### 2.2.3 Role → Alias → Device Mapping Chain

The hardware I/O system uses a three-level mapping chain:

```
role  →  deviceAlias  →  (deviceId, channelIndices)
```

This mirrors the plugin aliasing approach:

- **Roles** represent project-level intent (e.g., “main output should be the
  control room speakers”).
- **Aliases** provide machine-level realisation (e.g., “on this machine, the
  control room is `interface.studioMain:main`”).
- **Device mappings** are the hardware-level concretisation (e.g., “that alias
  resolves to device ID `0x1234`, channels 0–1”).

At runtime, Pulse/Aura can resolve:
- `role` → `deviceAlias` → actual device mapping

This allows projects to specify intent (roles) that adapt to different machines
via aliases, which in turn map to the actual hardware available on each machine.

Composer may assist in providing *recommended* alias/channel maps for known
devices, but the Hardware I/O system must work independently and offline.

### 2.3 Sample Rate and Buffer Size

Global audio stream configuration includes:

- **sample rate** (e.g. 44.1 kHz, 48 kHz, 96 kHz),
- **buffer size** (e.g. 64, 128, 256 samples).

Not all devices support arbitrary combinations. Signal is responsible for:

- querying supported rates/sizes,
- attempting configuration changes,
- reporting actual values in use (which may differ from requested values).

Pulse communicates:

- the current effective sample rate and buffer size,
- the last requested configuration,
- any mismatches or fallbacks.

### 2.4 Clock Source and Latency

Devices may have:

- different **clock sources** (internal, external word clock, ADAT, etc.),
- different **latency characteristics** (input, output, and round-trip latency).

Signal obtains latency information from the driver where possible.

Pulse exposes:

- effective clock source (where known),
- reported input/output/round-trip latency estimates.

Automation, processing cohort (PC) scheduling and anticipative rendering may use this
information to compensate for hardware latency where appropriate.

### 2.5 Machine Identity

Hardware I/O configuration is machine-specific. To support different hardware
configurations for the same project across different machines, the system uses
a **machine identifier** (`machineId`).

The `machineId` may be:

- a hashed MAC address,
- an OS-provided machine UUID,
- or another stable, local-only identifier.

The identifier is used **only** for differentiating machine-specific hardware
profiles. It is **not** used for licensing, DRM, or analytics.

Each machine maintains its own alias-to-device mappings, allowing the same
project to work seamlessly across different hardware setups while preserving
user preferences per machine.

---

## 3. Commands (Aura → Pulse)

Aura issues Hardware I/O commands when:

- the user selects or changes an audio device,
- the user changes sample rate/buffer size,
- channel mapping needs to be adjusted,
- the user requests diagnostics or a refresh.

Pulse validates requests, applies them via Signal, and emits events.

### 3.1 Device Discovery and Selection

#### **`hardware.listDevices`**

Request the current list of input and output devices.

Fields:
- (none; the request is implicit)

Pulse responds with a snapshot of known devices via `hardware.devicesUpdated`.

---

#### **`hardware.setActiveDevices`**

Select active input and/or output devices.

Fields include:

- `inputDeviceId` (optional),
- `outputDeviceId` (optional).

If a single duplex device is used, both may be the same.  
Pulse requests the change from Signal; the result is reported via
`hardware.streamConfigurationChanged` or an error event.

---

### 3.2 Stream Configuration

#### **`hardware.setStreamConfiguration`**

Request changes to:

- sample rate,
- buffer size,
- optional flags (e.g. “low latency” vs “safe” modes if exposed).

Fields:

- `sampleRate` (optional),
- `bufferSize` (optional),
- optional configuration hints.

Signal attempts to apply the configuration. Pulse then:

- updates its model with the actual configuration in use,
- emits `hardware.streamConfigurationChanged`,
- or emits an error event if the configuration cannot be applied.

Aura should treat this as “best effort” and display the effective values,
not just the requested ones.

---

### 3.3 Channel Mapping

#### **`hardware.setChannelMapping`**

Set the mapping between internal logical outputs (e.g. "Main out", "Cue 1") and
hardware device channels.

Fields include:

- `role` (logical role, e.g. `main`, `cue1`, `controlRoom`, etc.),
- optional `deviceAlias` (machine-specific alias for the device/channel group),
- per-output-channel assignments:

  - logical output slot → `deviceId`, `channelIndex`.

The mapping chain works as follows:

- `role` is stable across machines and represents project-level intent.
- `deviceAlias` is machine-specific and provides a human-friendly identifier
  for the device/channel group on this machine.
- `deviceId` and `channelIndices` are the physical endpoints resolved at
  runtime.

For example:

- On machine A, `role.main` may map to `deviceAlias: interface.studioMain:main`,
  which resolves to `deviceId: 0x1234, channelIndices: [0, 1]`.
- On machine B, the same `role.main` may map to `deviceAlias: interface.laptop:main`,
  which resolves to `deviceId: 0x5678, channelIndices: [0, 1]`.

Pulse/Aura can resolve `role` → `deviceAlias` → actual device mapping at
runtime. If a `deviceAlias` is provided, it is used to look up the current
device binding for that machine. If not provided, Pulse may attempt to match
by device capabilities, name, or vendor.

Input mappings (e.g. which hardware inputs feed which Tracks/Channels) are
primarily handled by the Routing and Channel domains, but Hardware I/O may
expose high-level mappings for core concepts like "default recording input".

Details of logical mapping identifiers are defined in the Routing and Channel
specs; this command provides the low-level hardware binding.

---

#### **`hardware.setDeviceAlias`**

Set or update a device alias mapping for the current machine.

Fields include:

- `deviceAlias` (the alias identifier),
- `deviceId` (the physical device ID),
- optional `channelGroup` (channel group identifier within the device),
- optional `channelIndices` (specific channel indices if applicable).

This command establishes or updates the machine-level binding from an alias to
a physical device/channel group. The binding is stored locally and persists
across sessions for this machine.

---

#### **`hardware.clearDeviceAlias`**

Remove a device alias mapping for the current machine.

Fields include:

- `deviceAlias` (the alias identifier to remove).

After clearing, any role mappings that referenced this alias will need to be
reconfigured or will fall back to device matching by other means.

---

#### **`hardware.setRoleMapping`** (optional)

Explicitly bind a role to a device alias for the current machine.

Fields include:

- `role` (the logical role),
- `deviceAlias` (the alias to bind to).

This provides an explicit role → alias binding. If not set, Pulse may infer
the mapping from channel mapping commands or attempt automatic matching.

---

### 3.4 Refresh and Diagnostics

#### **`hardware.refreshDevices`**

Hint to Pulse/Signal to rescan hardware devices:

- used after the user plugs or unplugs devices,
- or when the OS reports hardware changes.

Pulse responds with `hardware.devicesUpdated` once the scan completes.

---

#### **`hardware.requestDiagnostics`**

Request a diagnostic snapshot of the current hardware state.

Fields:
- optional flags (e.g. include latency details, include driver information).

Pulse responds via `hardware.diagnosticsUpdated`, which may include:

- device identifiers,
- current stream configuration,
- reported latencies,
- recent error flags (e.g. xruns).

---

## 4. Events (Pulse → Aura)

### 4.1 Device and Stream Events

#### **`hardware.devicesUpdated`**

Emitted when the device list changes or in response to `hardware.listDevices`
/ `hardware.refreshDevices`.

Includes:

- list of input devices:

  - `deviceId`, `name`, `vendor`,
  - flags (default, currently active),
  - capabilities (supported sample rates/buffer sizes where known),
  - channel counts and labels.

- list of output devices (same shape).

Aura uses this to populate device selectors.

---

#### **`hardware.streamConfigurationChanged`**

Emitted when the active stream configuration changes.

Includes:

- active `inputDeviceId`/`outputDeviceId`,
- `sampleRate` in use,
- `bufferSize` in use,
- reported latencies,
- optional driver or API metadata.

Aura updates its UI to reflect the **effective** configuration.

---

### 4.2 Channel Mapping Events

#### **`hardware.channelMappingChanged`**

Emitted when hardware channel mappings are updated.

Includes:

- mapping context (e.g. `main`, `cue1`, etc.),
- mappings from logical output slots to hardware device channels.

Aura uses this to render routing summaries and avoid desynchronised UI.

---

#### **`hardware.deviceAliasChanged`**

Emitted when a device alias mapping is created, updated, or removed.

Includes:

- `deviceAlias` (the alias identifier),
- `deviceId` (the current device ID, or null if cleared),
- optional `channelGroup` and `channelIndices`.

Aura uses this to update UI showing alias-to-device bindings and to refresh
role mapping displays.

---

#### **`hardware.roleMappingChanged`**

Emitted when a role → alias mapping is created, updated, or removed.

Includes:

- `role` (the logical role),
- `deviceAlias` (the current alias, or null if cleared).

Aura uses this to reflect role-to-alias bindings in the UI and to indicate
when roles need configuration on the current machine.

---

### 4.3 Error and Health Events

#### **`hardware.deviceError`**

Emitted when a device fails:

- device disconnected,
- driver error,
- stream start failure,
- other unrecoverable issues.

Includes:

- `deviceId` (where known),
- error code and message.

Aura should warn the user and potentially fall back to a different device or
to a “no audio” state.

---

#### **`hardware.xrunDetected`**

Reports buffer underruns/overruns (xruns).

Fields:

- count or recent rate,
- associated device/stream.

Aura may display this in diagnostics or status indicators.

---

#### **`hardware.diagnosticsUpdated`**

Response to `hardware.requestDiagnostics`, summarising the current state.

---

## 5. Snapshot and Persistence Semantics

Hardware I/O configuration uses a two-tier persistence model, mirroring the
approach used in window layout architecture:

### 5.1 Project-Level Persistence

Project files store **role → deviceAlias** preferences. This represents *user
intent* (e.g., "main output should be the control room speakers") and is
portable across machines.

Project files should store:

- role → deviceAlias mappings (e.g., `role.main` → `interface.studioMain:main`),
- preferred sample rate and buffer size,
- logical channel mapping intentions expressed as roles.

Project files should **not** store device IDs, as those vary by machine and
are not portable.

### 5.2 Machine-Level Persistence

Machine-level storage (outside the project, keyed by `machineId`) stores:

- **deviceAlias → (deviceId, channelIndices)** bindings,
- supported buffer sizes and sample rates for each device,
- preferred defaults for the machine,
- per-ScreenSet preferences, if relevant.

This information is stored locally and is **not** included in project files.
Each machine maintains its own alias-to-device mappings, allowing the same
project to work seamlessly across different hardware setups.

### 5.3 Loading Logic

When a project opens:

1. Compute `machineId` for the current machine.
2. Determine available devices on this machine.
3. For each project-specified role:

   - If the role's `deviceAlias` exists on this machine → resolve to
     device+channels using the machine-level binding.
   - If the alias does not exist → attempt to:

     - match by device capabilities (e.g., stereo output, sample rate support),
     - match by name or vendor (fuzzy matching),
     - fall back safely and notify the user that configuration is needed.

4. Apply machine-level defaults for sample rate and buffer size, or use
   project preferences if the machine supports them.

This approach ensures that projects express intent (roles and aliases) while
allowing each machine to maintain its own hardware-specific bindings.

### 5.4 Snapshot Application

When Aura receives a `project.snapshot` that includes this domain, it MUST treat
the snapshot as the **authoritative** representation of the current state for
this domain.

In particular:

- Snapshot data replaces any locally cached state in Aura for this domain.

- Aura MUST discard or reconcile any pending local edits that conflict with the
  snapshot according to its own UX rules (for example, by cancelling unsent
  edits, or prompting the user where appropriate).

- Incremental events for this domain (e.g. `*.added`, `*.updated`,
  `*.removed`) are only applied on top of the last successfully applied
  snapshot.

Snapshots are **replacements**, not merges. If Aura is unsure whether to trust
a local view or a snapshot, it must prefer the snapshot.

Pulse is responsible for ensuring that snapshots across all domains are
internally consistent at the moment they are emitted.

Snapshots (as used for undo/redo or historical states) generally do **not**
include hardware configuration; they operate over the internal audio graph
and project model, not external devices.

---

## 6. Realtime and Cohort Considerations

Hardware I/O configuration changes can be disruptive and must be handled
carefully:

- Signal must reconfigure devices without blocking the audio thread longer
  than necessary.
- Some changes may require stopping and restarting the engine (e.g. changing
  sample rate on certain drivers).
- Pulse may need to:

  - temporarily pause transport,
  - invalidate anticipative buffers,
  - recompute timing/latency assumptions.

Cohort-related logic:

- Latency reports influence scheduling and automation timing.
- Device changes may temporarily force Nodes into a safe/live cohort until
  the new configuration is stable.

The Hardware I/O domain itself is not executed on the audio thread, but its
effects propagate to the engine’s realtime behaviour. Implementation must
respect the realtime constraints described in the realtime safety guidelines.

---

## 7. Relationship to Other Domains

The Hardware I/O domain interacts with:

- **Signal architecture**  
  The Signal layer contains the actual audio backend and driver interfaces.
  Hardware I/O commands are realised there.

- **Routing and Channel domains**  
  Logical routing from Channels to hardware outputs and from hardware inputs
  into Channels is defined by Routing/Channel IPC. Hardware I/O provides the
  available endpoints.

- **Tempo and Transport**  
  Device changes may affect timing; transport may need to be stopped or
  restarted during reconfiguration.

- **Processing Cohorts**  
  Hardware latency and buffer size influence cohort scheduling and
  anticipative render horizon.

- **Window Layout and Diagnostics UIs**  
  Device status, xruns and diagnostics will be surfaced in Aura’s settings
  and status panels, using information from this domain.

The Hardware I/O domain provides a stable abstraction over platform-specific
audio APIs, allowing Aura and Pulse to reason about devices without embedding
OS-level details.

**Relationship to Control Surfaces:** Control surfaces and MIDI controllers are handled separately. Pulse maintains mapping rules and context-aware behaviour for control surfaces, while Signal handles low-level device enumeration and I/O transport. Pulse sends mapping decisions and Feedback Intents to Signal, not Aura. Mapping rules operate on abstract controls (pads/faders/buttons/etc.) and ParameterIds/ActionIds.
