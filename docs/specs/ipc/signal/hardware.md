# Signal IPC – Hardware Domain

This document defines the **Hardware** IPC domain between Pulse and Signal.

The Hardware domain is responsible for:

- discovering and describing **audio** and **MIDI** devices,
- selecting and configuring the active audio interface(s),
- reporting device state changes (connected/disconnected, errors),
- exposing external sync capabilities (MIDI clock, MTC/LTC, Link-style sync).

Pulse owns:

- mapping physical devices to **logical roles** (e.g. “Main Out”, “Headphones”),
- persistence and aliasing of devices across machines and sessions,
- decisions about which devices to use in a given project,
- integration with Composer’s hardware metadata/telemetry.

Signal owns:

- low-level interaction with OS APIs (CoreAudio, ASIO, WASAPI, ALSA, etc.),
- ensuring that hardware configuration is consistent with engine configuration,
- detecting and reporting device-related problems.

Aura never talks directly to Signal; all commands flow **Pulse → Signal**.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Hardware domain:

- enumerates audio input and output devices and their capabilities,
- enumerates MIDI input/output ports,
- applies requested audio device selections and configurations:
  - chosen input device,
  - chosen output device,
  - buffer size,
  - sample rate (where supported),
- reports:
  - device add/remove events,
  - device lost / recovered,
  - errors and warnings (dropouts, configuration issues),
- exposes external sync endpoints (MIDI clock, MTC, LTC, Link-style, etc.).

### 1.2 Non-Goals

The Hardware domain does **not**:

- manage audio routing within the DAW (that is handled by `signal.graph` and
  Pulse’s routing model),
- decide which device “role” (e.g. “Control Room”, “Headphones”) a hardware
  channel corresponds to,
- manage recording destinations (that belongs to recording/media specs),
- implement per-plugin audio/MIDI routing (that belongs to the graph / node
  model).

---

## 2. Identity & Stable Hardware IDs

Hardware devices are inherently volatile:

- devices can be unplugged,
- OS identifiers can change across reboots or driver reinstalls.

To support stable configuration and Composer integration, Signal provides:

- a **stable-ish hardware ID** per device, and
- detailed raw OS identifiers for fallback.

Example Audio Device identity:

```json
{
  "deviceId": "audio:mac:CoreAudio:XYZ123",
  "osDeviceId": "CoreAudioDevice:0x00112233",
  "name": "RME Babyface Pro FS",
  "manufacturer": "RME"
}
```

Pulse will:

- store `deviceId` in project/session preferences,
- map those to logical roles,
- use Composer to learn which devices are often used for which roles,
- handle remapping when a given `deviceId` is not present.

Signal is responsible only for:

- generating consistent `deviceId` values on a given machine as far as
  possible,
- exposing enough OS identifiers to allow Pulse/Composer to reason about
  equivalence and substitutions.

---

## 3. Message Flow Overview

Typical flows:

1. On startup or device change, Pulse calls `hardware.listAudioDevices` and
   `hardware.listMidiDevices`.
2. Pulse chooses input/output devices and calls `hardware.setAudioConfig`.
3. Signal:
   - applies the configuration,
   - updates the engine’s effective sample rate and block size,
   - informs Pulse via Engine and Hardware events.
4. If devices are unplugged or become unavailable:
   - Signal emits `hardware.audioDeviceLost` / `hardware.midiDeviceLost`,
   - Pulse remaps roles or prompts the user,
   - Pulse may call `hardware.setAudioConfig` again with new devices.

---

## 4. Commands (Pulse → Signal)

### 4.1 `hardware.listAudioDevices`

List available audio input and output devices.

**Request**

```json
{
  "command": "hardware.listAudioDevices"
}
```

**Response**

```json
{
  "replyTo": "hardware.listAudioDevices",
  "devices": [
    {
      "deviceId": "audio:mac:CoreAudio:XYZ123",
      "osDeviceId": "CoreAudioDevice:0x00112233",
      "name": "RME Babyface Pro FS",
      "manufacturer": "RME",
      "isDefaultInput": true,
      "isDefaultOutput": true,
      "inputChannels": 8,
      "outputChannels": 8,
      "sampleRates": [44100, 48000, 96000],
      "preferredSampleRate": 48000,
      "bufferSizes": [64, 128, 256, 512, 1024],
      "minBufferSize": 64,
      "maxBufferSize": 2048,
      "canChangeSampleRate": true
    }
  ]
}
```

Semantics:

- Device list reflects the current OS state.
- `sampleRates` and `bufferSizes` are indicative; some APIs expose ranges rather
  than discrete lists, which Signal may approximate.
- Pulse uses this to:
  - populate device selection UI,
  - choose defaults,
  - map devices to logical roles.

---

### 4.2 `hardware.listMidiDevices`

List available MIDI input and output ports.

**Request**

```json
{
  "command": "hardware.listMidiDevices"
}
```

**Response**

```json
{
  "replyTo": "hardware.listMidiDevices",
  "inputs": [
    {
      "portId": "midiIn:mac:0",
      "name": "Keystep 37",
      "manufacturer": "Arturia",
      "supportsMPE": false
    }
  ],
  "outputs": [
    {
      "portId": "midiOut:mac:0",
      "name": "Keystep 37",
      "manufacturer": "Arturia"
    }
  ]
}
```

Semantics:

- Pulse may subscribe Tracks/Controllers to these ports via separate MIDI/Control
  IPC domains (to be defined).
- `supportsMPE` is a hint; detailed MPE/profile information may be added later.

---

### 4.3 `hardware.setAudioConfig`

Select and configure the active audio devices and basic engine hardware config.

**Request**

```json
{
  "command": "hardware.setAudioConfig",
  "inputDeviceId": "audio:mac:CoreAudio:XYZ123",
  "outputDeviceId": "audio:mac:CoreAudio:XYZ123",
  "sampleRate": 48000,
  "bufferSize": 256,
  "options": {
    "exclusiveMode": true,
    "lowLatencyMode": true
  }
}
```

Semantics:

- Pulse may choose the same device for input and output, or different ones
  where the OS supports it.
- `sampleRate` and `bufferSize` are **desired** values:
  - the OS/driver may round or override them,
  - effective values are reported back via Engine (`engine.configResolved`) and
    Hardware events (`hardware.audioConfigChanged`).
- `exclusiveMode` and `lowLatencyMode` are hints; support depends on platform.

---

### 4.4 `hardware.getAudioConfig`

Query the current effective audio hardware configuration.

**Request**

```json
{
  "command": "hardware.getAudioConfig"
}
```

**Response**

```json
{
  "replyTo": "hardware.getAudioConfig",
  "config": {
    "inputDeviceId": "audio:mac:CoreAudio:XYZ123",
    "outputDeviceId": "audio:mac:CoreAudio:XYZ123",
    "sampleRate": 47999.8,
    "bufferSize": 256,
    "options": {
      "exclusiveMode": true,
      "lowLatencyMode": true
    }
  }
}
```

---

### 4.5 `hardware.subscribeMidiInput`

Subscribe an endpoint in Pulse to MIDI events from a given MIDI input port.

(This is a very lightweight description; the full MIDI IPC domain will carry
more detail. Here we only cover the binding.)

**Request**

```json
{
  "command": "hardware.subscribeMidiInput",
  "portId": "midiIn:mac:0",
  "subscriptionId": "midiSub:pulse:track:42"
}
```

Semantics:

- `subscriptionId` is chosen by Pulse.
- Signal will:
  - listen to the given `portId`,
  - emit MIDI events addressed to `subscriptionId` via a dedicated MIDI/Control
    IPC domain (specified elsewhere).
- Unsubscribe is done via `hardware.unsubscribeMidiInput`.

---

### 4.6 `hardware.unsubscribeMidiInput`

Cancel a MIDI input subscription.

**Request**

```json
{
  "command": "hardware.unsubscribeMidiInput",
  "subscriptionId": "midiSub:pulse:track:42"
}
```

---

### 4.7 `hardware.setSyncConfig`

Configure external sync behaviour (tempo sync, timecode, etc).

**Request**

```json
{
  "command": "hardware.setSyncConfig",
  "sync": {
    "midiClockOut": {
      "enabled": false,
      "portId": null
    },
    "mtcOut": {
      "enabled": false,
      "portId": null
    },
    "ltcIn": {
      "enabled": false,
      "deviceId": null
    },
    "linkLikeSync": {
      "enabled": false
    }
  }
}
```

Semantics:

- Detailed behaviour (e.g. Link-like protocols) is implementation-specific and
  can be expanded in future specs.
- Transport’s position is used as the master time for these sync signals unless
  configured otherwise.

---

## 5. Events (Signal → Pulse)

### 5.1 `hardware.audioDevicesChanged`

The set of available audio devices has changed.

**Payload**

```json
{
  "event": "hardware.audioDevicesChanged",
  "devices": [
    {
      "deviceId": "audio:mac:CoreAudio:XYZ123",
      "osDeviceId": "CoreAudioDevice:0x00112233",
      "name": "RME Babyface Pro FS",
      "manufacturer": "RME",
      "isDefaultInput": true,
      "isDefaultOutput": true,
      "inputChannels": 8,
      "outputChannels": 8,
      "sampleRates": [44100, 48000, 96000],
      "preferredSampleRate": 48000,
      "bufferSizes": [64, 128, 256, 512, 1024],
      "minBufferSize": 64,
      "maxBufferSize": 2048,
      "canChangeSampleRate": true
    }
  ]
}
```

Semantics:

- Emitted when devices are added/removed or their properties change.
- Pulse may:
  - reconcile persisted `deviceId` mappings,
  - update project preferences,
  - prompt the user if the active config is affected.

---

### 5.2 `hardware.audioDeviceLost`

The currently active audio device(s) have been lost or become unusable.

**Payload**

```json
{
  "event": "hardware.audioDeviceLost",
  "inputDeviceId": "audio:mac:CoreAudio:XYZ123",
  "outputDeviceId": "audio:mac:CoreAudio:XYZ123",
  "message": "The selected audio device was disconnected."
}
```

Semantics:

- Pulse must treat this as a serious event:
  - mark the current configuration as invalid,
  - either:
    - attempt automatic fallback (e.g. system default device) by calling
      `hardware.setAudioConfig`, or
    - present a device-selection UI to the user.
- Engine will likely enter an error or stopped state; `engine.stateChanged` and
  `engine.fatalError` may also be emitted.

---

### 5.3 `hardware.audioConfigChanged`

The effective audio configuration has changed.

**Payload**

```json
{
  "event": "hardware.audioConfigChanged",
  "config": {
    "inputDeviceId": "audio:mac:CoreAudio:XYZ123",
    "outputDeviceId": "audio:mac:CoreAudio:XYZ123",
    "sampleRate": 48000,
    "bufferSize": 256,
    "options": {
      "exclusiveMode": true,
      "lowLatencyMode": true
    }
  }
}
```

Semantics:

- May be emitted:
  - after a successful `hardware.setAudioConfig`,
  - after OS-level changes, e.g. user tweaks device settings outside Loophole.
- Pulse should:
  - treat this as authoritative,
  - update its notion of project sample rate,
  - ensure transport/timebase alignment.

---

### 5.4 `hardware.midiDevicesChanged`

The set of MIDI ports has changed.

**Payload**

```json
{
  "event": "hardware.midiDevicesChanged",
  "inputs": [
    {
      "portId": "midiIn:mac:0",
      "name": "Keystep 37",
      "manufacturer": "Arturia",
      "supportsMPE": false
    }
  ],
  "outputs": [
    {
      "portId": "midiOut:mac:0",
      "name": "Keystep 37",
      "manufacturer": "Arturia"
    }
  ]
}
```

---

### 5.5 `hardware.midiDeviceLost`

A MIDI port previously in use has been lost.

**Payload**

```json
{
  "event": "hardware.midiDeviceLost",
  "portId": "midiIn:mac:0",
  "message": "MIDI device was disconnected."
}
```

Semantics:

- Pulse should:
  - mark relevant subscriptions as inactive,
  - optionally prompt the user or auto-remap when possible.

---

### 5.6 `hardware.warning`

Non-fatal hardware-related issues.

**Payload**

```json
{
  "event": "hardware.warning",
  "code": "highLatency",
  "message": "Audio buffer size is high; latency may be noticeable.",
  "details": {
    "bufferSize": 2048,
    "sampleRate": 44100
  }
}
```

---

### 5.7 `hardware.error`

Hardware-related errors that may not be fatal to the engine, but require
attention.

**Payload**

```json
{
  "event": "hardware.error",
  "code": "deviceConfigFailed",
  "message": "Unable to apply requested audio configuration.",
  "details": {
    "inputDeviceId": "audio:mac:CoreAudio:XYZ123",
    "outputDeviceId": "audio:mac:CoreAudio:XYZ123"
  }
}
```

Semantics:

- Pulse should treat these as authoritative, and may:
  - rollback to a previous configuration,
  - present options to the user.

---

## 6. Error Handling

Hardware commands can fail synchronously for reasons including:

- device ID not found,
- invalid sample rate or buffer size,
- OS-level API failure,
- permission issues (e.g. macOS microphone permission denied).

Example error:

```json
{
  "error": {
    "code": "deviceNotFound",
    "message": "Requested audio device was not found.",
    "domain": "hardware",
    "details": {
      "deviceId": "audio:mac:CoreAudio:XYZ123"
    }
  }
}
```

Pulse must treat error responses as authoritative and avoid making assumptions
about the underlying hardware state.

---

## 7. Relationship to Other Signal Domains

- **signal.engine**  
  Effective hardware configuration directly impacts engine configuration:
  - sample rate,
  - block size,
  - channel counts.

- **signal.transport**  
  Transport operates on sample-based positions; these depend on the effective
  sample rate from hardware/engine.

- **signal.graph**  
  Graph outputs (e.g. master Channel) ultimately connect to hardware outputs
  exposed by this domain.

- **signal.diagnostics**  
  Hardware issues (dropouts, high latency, device flapping) should be reflected
  in Diagnostics metrics and events.

---

## 8. Extensibility

Future additions may include:

- aggregate device handling (combining multiple devices into one virtual
  device),
- advanced clock sync configuration (word clock, audio interfaces with multiple
  clock sources),
- per-channel calibration and latency compensation metadata,
- deeper integration with Composer for:
  - shared hardware profiles,
  - cross-machine mappings,
  - user preferences about which devices are preferred for which roles.

The Hardware domain should remain a **thin, OS-facing layer**, with all
long-lived mapping, aliasing, and role assignment logic residing in Pulse and
Composer.
