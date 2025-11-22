# Inter-Process Communication Overview

This document defines the communication model between the three primary Loophole
processes: Aura, Pulse and Signal. It establishes the conceptual structure of
IPC in Loophole, the division of domains and responsibilities, and the
distinction between the different communication planes used by the system.

This is a **guidelines document**. Specific message types, schemas and framing
details are defined in the corresponding specification files within
`@chorus:/docs/specs/ipc/`.

---

## Contents

- [1. Goals](#1-goals)
- [2. Architectural Roles](#2-architectural-roles)
  - [2.1 Aura (UI Layer)](#21-aura-ui-layer)
  - [2.2 Pulse (Project and Data Model)](#22-pulse-project-and-data-model)
  - [2.3 Signal (Audio Engine)](#23-signal-audio-engine)
- [3. Traffic Classes](#3-traffic-classes)
  - [3.1 Control and Model Plane](#31-control-and-model-plane)
  - [3.2 Engine Stream Plane](#32-engine-stream-plane)
  - [3.3 Telemetry Plane](#33-telemetry-plane)
- [4. Routing Model](#4-routing-model)
  - [4.1 Aura to Pulse](#41-aura-to-pulse)
  - [4.2 Pulse to Signal](#42-pulse-to-signal)
  - [4.3 Signal to Pulse](#43-signal-to-pulse)
  - [4.4 Signal to Aura](#44-signal-to-aura)
- [5. Pulse IPC Domains](#5-pulse-ipc-domains)
  - [5.1 Core Project Domains](#51-core-project-domains)
  - [5.2 Media and Recording](#52-media-and-recording)
  - [5.3 Plugin Management](#53-plugin-management)
  - [5.4 Engine and Diagnostics](#54-engine-and-diagnostics)
  - [5.5 System and Metadata](#55-system-and-metadata)
- [6. Processing Cohorts and Anticipative Rendering](#6-processing-cohorts-and-anticipative-rendering)
- [7. Parameter Gestures and Streaming](#7-parameter-gestures-and-streaming)
  - [7.1 Begin and End Messages](#71-begin-and-end-messages)
  - [7.2 High-Rate Parameter Streaming](#72-high-rate-parameter-streaming)
  - [7.3 Model Commitment](#73-model-commitment)
- [8. Early-Stage Implementation Notes](#8-early-stage-implementation-notes)
- [9. Hardware Domains](#9-hardware-domains)
- [10. Future Process Separation](#10-future-process-separation)

---

## 1. Goals

Loophole uses multiple processes to separate concerns, ensure real-time
performance, preserve stability when hosting third-party plugins, and allow
future distribution of work across languages and machines.

The IPC design aims to:

- Preserve **Pulse** as the authoritative owner of project state.
- Keep **Signal** isolated and responsible only for real-time audio/MIDI and
  plugin execution.
- Keep **Aura** focused on user interaction and presentation.
- Avoid routing high-frequency engine streams through Pulse.
- Allow Pulse to move out-of-process later without changes to Aura or Signal.
- Support efficient high-frequency parameter updates and telemetry without
  overwhelming JSON processing.

---

## 2. Architectural Roles

### 2.1 Aura (UI Layer)

Aura is the graphical interface of Loophole. It presents the project state,
collects user intent, and displays real-time telemetry from Signal.

Aura does **not** own project data, engine configuration, or plugin state. It
derives all such information from Pulse.

### 2.2 Pulse (Project and Data Model)

Pulse owns and manages all project state. It validates, applies and persists all
editing operations. It is responsible for deriving real-time-safe engine
structures that Signal can execute.

Pulse interprets user intent, updates the model and issues engine commands based
on model changes. Pulse is therefore the **hub** of the control and model plane.

### 2.3 Signal (Audio Engine)

Signal executes graphs derived by Pulse, hosts plugins, produces telemetry and
reports engine and hardware state. Signal is authoritative for real-time
behaviour but not for persistent project data.

---

## 3. Traffic Classes

Loophole divides IPC traffic into three logical planes. These planes may share
transport in early versions but are conceptually distinct.

### 3.1 Control and Model Plane

This plane carries:

- Editing operations (commands from Aura to Pulse)
- Project state changes (events from Pulse to Aura, correlated to commands via `cid`)
- Graph updates
- Engine control messages (commands from Pulse to Signal)
- Engine state notifications relevant to the model (events from Signal to Pulse)

Traffic on this plane is structured, low-frequency and encoded as JSON envelopes.
Pulse is the hub of this plane.

### 3.2 Engine Stream Plane

This plane carries high-frequency control messages required for real-time
interaction, such as continuous parameter changes during a user gesture.

Characteristics:

- High-rate, small messages
- Compact representation (binary or minimal framed format)
- Routed directly between Pulse and Signal (Signal ↔ Pulse only)
- Pulse aggregates and forwards curated updates to Aura via standard IPC events

### 3.3 Telemetry Plane

This plane carries high-frequency engine output used for UI display:

- Meters
- Spectrum analysis
- Transport timing
- Other visualisation-oriented signals

Telemetry flows from Signal to Pulse via a high-rate channel (Signal ↔ Pulse only). Pulse aggregates this data and forwards curated updates to Aura via standard JSON IPC events. Aura never establishes a direct connection to Signal.

---

## 4. Routing Model

### 4.1 Pulse-Centred Routing

All high-level commands originate from Aura and go to Pulse. Pulse is the
central router/authority for project state and model changes:

- **Aura → Pulse**: all project-level commands (open, save, track edits, etc.)
- **Pulse → Aura**: project snapshots and model-change events, aggregated metering/analysis data
- **Pulse → Signal**: graph rebuild instructions and engine commands, aggregated gesture streams
- **Signal → Pulse**: engine status, errors, confirmations, and high-rate metering/analysis streams (never initiates project-level changes)

Signal and Composer never communicate directly with Aura. Signal and Composer never communicate directly with each other. All model state flows through Pulse. Any side-channel (binary or high-rate) is strictly **Signal ↔ Pulse**, not Aura.

### 4.2 Aura to Pulse

Aura translates user interaction into high-level project intent:

- Track creation and deletion
- Plugin insertion and removal
- Editing actions
- Transport intent
- Parameter gesture start/end (but not the high-rate stream)

Pulse validates these commands and updates the model.

### 4.3 Pulse to Signal

Pulse issues real-time-safe engine instructions:

- Graph replacement or patch
- Plugin lifecycle operations
- Parameter set commands
- Transport control

Pulse is the sole originator of structural changes to the engine graph.

### 4.4 Signal to Pulse

Signal reports engine and hardware events relevant to the project model:

- Plugin load failures
- Device availability changes
- Latency or sample rate changes
- Graph application results

Pulse updates the model and emits model-change events to Aura.

### 4.5 Signal Communication with Aura

**Aura never establishes any IPC connection to Signal.**

All communication flows through Pulse:

- **Signal → Pulse**: high-rate metering/analysis streams (via side-channel if needed)
- **Pulse → Aura**: aggregated and curated metering/analysis events (via standard JSON IPC)

Aura receives all engine analysis, metering and diagnostic data from Pulse via standard IPC envelopes. Pulse aggregates high-rate streams from Signal and forwards them as JSON IPC events to clients (e.g., Aura).

---

## 5. Pulse IPC Domains

Pulse IPC is organised into domain-specific namespaces, each covering a distinct
area of functionality. Key domains include:

### 5.1 Core Project Domains

- **`project`** — Project lifecycle (open, save, new, close)
- **`transport`** — Playback control and position
- **`timebase`** — Tempo, time signatures, and timing
- **`track`** — Track creation, deletion, and organisation
- **`clip`** — Audio and MIDI clip management
- **`lane`** — Clip lane organisation within tracks
- **`channel`** — Channel strip configuration and routing
- **`node`** — DSP graph node management
- **`parameter`** — Parameter values and automation
- **`automation`** — Automation curve editing
- **`routing`** — Signal routing and send/return paths

### 5.2 Media and Recording

- **`media`** — Media file management and metadata
- **`recording`** — Audio and MIDI recording operations
- **`rendering`** — Offline rendering and export

### 5.3 Plugin Management

- **`pluginLibrary`** — Plugin organisation, search, tagging, favourites, chains,
  presets, and high-level insert/replace operations (Aura ↔ Pulse only)
- **`pluginUi`** — Plugin UI window lifecycle and geometry management

See `docs/specs/ipc/pulse/plugin-library.md` for the Plugin Library domain
specification, and `docs/architecture/33-plugin-library-and-browser.md` for the
architectural overview.

### 5.4 Engine and Diagnostics

- **`engine`** — Engine configuration and control
- **`cohort`** — Processing cohort assignment
- **`engineDiagnostics`** — Performance monitoring and diagnostics
- **`metering`** — Audio level metering
- **`gesture`** — Parameter gesture boundaries

### 5.5 System and Metadata

- **`session`** — Session management and background tasks
- **`hardwareIo`** — Audio hardware device configuration
- **`history`** — Undo/redo history
- **`projectMetadata`** — Project-level metadata and markers
- **`ui`** — General UI state and layout

Each domain defines its own command and event names. Domain-specific
specifications are located in `docs/specs/ipc/pulse/` and `docs/specs/ipc/signal/`.

---

## 6. Processing Cohorts and Anticipative Rendering

IPC messages remain domain-based (Project, Transport, Track, etc.), but some
domains now have **engine-level consequences** for processing cohorts and
anticipative buffers:

- **Cohort decisions** are made entirely in Pulse and sent to Signal as:
  - graph updates
  - node annotations
- **Project load operations** (domain: `project`, name: `open` or `new`) trigger:
  - fresh cohort assignment in Pulse
  - graph rebuild instructions to Signal
- **Transport operations** (seek, loop changes) may trigger:
  - anticipative buffer invalidation
  - render horizon rebuilds

The IPC command schemas themselves do not change; the dual-engine architecture
(Anticipative and Live engines) affects how Signal internally processes these
commands, not the IPC message structure.

---

## 7. Parameter Gestures and Streaming

Continuous parameter interaction requires a separation between:

- High-frequency real-time updates for the engine
- Lower-frequency semantic updates for the project model

This is handled through **parameter gestures**.

### 7.1 Begin and End Messages

Aura notifies Pulse when a user begins a gesture:

```
domain: "parameter", name: "beginGesture"
domain: "parameter", name: "endGesture"
```

Pulse updates its internal state and may generate semantic engine commands (such
as setting an initial or final value).

### 7.2 High-Rate Parameter Streaming

During the gesture, Pulse forwards compact high-frequency parameter updates to Signal via the engine stream plane.

The data-plane channel is strictly **Pulse ↔ Signal**. These messages are not JSON. Signal applies them immediately to the relevant plugin parameter.

Pulse aggregates gesture data from Aura and forwards it to Signal. Streams are identified with a gesture or stream identifier managed by Pulse.

### 7.3 Model Commitment

When the gesture ends, Aura sends a message with `domain: "parameter"`, `name: "endGesture"` to Pulse
including the final parameter value and optional automation data.

Pulse updates the project model and issues any necessary engine commands to
ensure consistency.

---

## 8. Early-Stage Implementation Notes

Initially, Pulse resides inside Aura. Despite this, Aura must treat Pulse as a
logical external service:

- Aura sends commands to Pulse via a defined interface.
- Pulse sends model-change events back to Aura.
- Pulse uses an EnginePort interface to send engine instructions to Signal.

Aura never generates engine commands directly.

As Pulse moves out-of-process, the in-process interfaces are replaced with IPC
adapters without changes to semantics.

---

## 9. Hardware Domains

Loophole's IPC includes hardware-related domains for both audio I/O and control surfaces:

### 9.1 Audio Hardware I/O

The Hardware I/O domain (Pulse ↔ Signal, Pulse ↔ Aura) handles:

- audio device enumeration and selection,
- sample rate and buffer size configuration,
- channel mapping and routing,
- latency reporting and clock source management.

Signal owns low-level audio API interaction; Pulse maintains stable configuration models.

### 9.2 Control Surfaces

Control surface communication follows a three-layer model:

- **Hardware discovery (Signal)**: Signal enumerates control surface devices (MIDI, HID, OSC) and emits device descriptors to Pulse.

- **Mapping and context (Pulse)**: Pulse maintains mapping rules, evaluates control inputs into control intents, generates feedback intents, and manages context-aware behaviour.

- **Device intelligence (Composer)**: Composer provides device profiles and default mapping recommendations based on device fingerprints from Pulse.

Signal acts as a transport layer: it parses low-level inputs and forwards structured control messages to Pulse, and sends hardware outputs (LEDs, displays) as requested by Pulse. Pulse owns all mapping logic and context-aware behaviour.

---

## 10. Future Process Separation

When Pulse becomes a standalone service:

- Aura connects to Pulse using the same project-protocol messages.
- Pulse connects directly to Signal for the control/model plane, engine stream plane, and telemetry plane.
- All high-rate streams (gesture, metering, analysis) flow through Pulse.
- Aura never establishes a direct IPC connection to Signal.

The message contracts defined here remain stable regardless of the process topology. Signal communicates exclusively with Pulse. Aura never opens a transport to Signal; all analysis, metering and gesture information is proxied by Pulse.
