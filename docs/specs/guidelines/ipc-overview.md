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
- [5. Parameter Gestures and Streaming](#5-parameter-gestures-and-streaming)
  - [5.1 Begin and End Messages](#51-begin-and-end-messages)
  - [5.2 High-Rate Parameter Streaming](#52-high-rate-parameter-streaming)
  - [5.3 Model Commitment](#53-model-commitment)
- [6. Early-Stage Implementation Notes](#6-early-stage-implementation-notes)
- [7. Future Process Separation](#7-future-process-separation)

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

- Editing operations
- Project state changes
- Graph updates
- Engine control messages
- Engine state notifications relevant to the model

Traffic on this plane is structured, low-frequency and encoded as JSON envelopes.
Pulse is the hub of this plane.

### 3.2 Engine Stream Plane

This plane carries high-frequency control messages required for real-time
interaction, such as continuous parameter changes during a user gesture.

Characteristics:

- High-rate, small messages
- Compact representation (binary or minimal framed format)
- Routed directly between Aura and Signal
- Pulse sees only gesture boundaries, not individual samples

### 3.3 Telemetry Plane

This plane carries high-frequency engine output used for UI display:

- Meters
- Spectrum analysis
- Transport timing
- Other visualisation-oriented signals

Telemetry is routed directly from Signal to Aura. Pulse does not process this
plane.

---

## 4. Routing Model

### 4.1 Aura to Pulse

Aura translates user interaction into high-level project intent:

- Track creation and deletion
- Plugin insertion and removal
- Editing actions
- Transport intent
- Parameter gesture start/end (but not the high-rate stream)

Pulse validates these commands and updates the model.

### 4.2 Pulse to Signal

Pulse issues real-time-safe engine instructions:

- Graph replacement or patch
- Plugin lifecycle operations
- Parameter set commands
- Transport control

Pulse is the sole originator of structural changes to the engine graph.

### 4.3 Signal to Pulse

Signal reports engine and hardware events relevant to the project model:

- Plugin load failures
- Device availability changes
- Latency or sample rate changes
- Graph application results

Pulse updates the model and emits model-change events to Aura.

### 4.4 Signal to Aura

High-rate telemetry bypasses Pulse entirely:

- Metering
- Transport state
- Spectral data

Aura uses this data for visual display.

---

## 5. Parameter Gestures and Streaming

Continuous parameter interaction requires a separation between:

- High-frequency real-time updates for the engine
- Lower-frequency semantic updates for the project model

This is handled through **parameter gestures**.

### 5.1 Begin and End Messages

Aura notifies Pulse when a user begins a gesture:

```
parameter.gesture.begin
parameter.gesture.end
```

Pulse updates its internal state and may generate semantic engine commands (such
as setting an initial or final value).

### 5.2 High-Rate Parameter Streaming

During the gesture, Aura sends compact high-frequency parameter updates directly
to Signal via the engine stream plane.

These messages are not JSON and do not involve Pulse. Signal applies them
immediately to the relevant plugin parameter.

Streams may be identified with a gesture or stream identifier provided by Pulse.

### 5.3 Model Commitment

When the gesture ends, Aura sends a `parameter.gesture.end` message to Pulse
including the final parameter value and optional automation data.

Pulse updates the project model and issues any necessary engine commands to
ensure consistency.

---

## 6. Early-Stage Implementation Notes

Initially, Pulse resides inside Aura. Despite this, Aura must treat Pulse as a
logical external service:

- Aura sends commands to Pulse via a defined interface.
- Pulse sends model-change events back to Aura.
- Pulse uses an EnginePort interface to send engine instructions to Signal.

Aura never generates engine commands directly.

As Pulse moves out-of-process, the in-process interfaces are replaced with IPC
adapters without changes to semantics.

---

## 7. Future Process Separation

When Pulse becomes a standalone service:

- Aura connects to Pulse using the same project-protocol messages.
- Pulse connects directly to Signal for the control/model plane.
- The engine stream and telemetry planes may remain between Aura and Signal or
  be re-evaluated depending on performance characteristics.

The message contracts defined here remain stable regardless of the process
topology.
