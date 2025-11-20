# Pulse Gesture Domain Specification

This document defines the Gesture domain of the Pulse project protocol.  
Gesture streams represent **high-rate, low-latency control channels** used to
drive parameters in real time — faders, knobs, MIDI controllers, touch/pen
gestures, XY pads, control surfaces, etc.

Critically, the Gesture domain is explicitly designed with a **dual-path model**:

- **Control-plane (Aura ↔ Pulse)**  
  Manages gesture stream lifecycle, binding, metadata, and precedence rules.
- **Data-plane (Aura → Signal)**  
  Carries high-frequency gesture samples directly to the audio engine.

Pulse *knows about* all gesture activity, but does not need to relay every
sample. Signal is the primary consumer of the high-rate control stream.

Final parameter state and low-rate updates are synchronised back to Pulse so
the model remains coherent and automation semantics are correct.

This design is mandatory and foundational — not optional or deferred.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Gesture Model](#2-gesture-model)  
  - [2.1 Control-Plane vs Data-Plane](#21-control-plane-vs-data-plane)  
  - [2.2 Streams and Targets](#22-streams-and-targets)  
  - [2.3 Sample Format and Reliability](#23-sample-format-and-reliability)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Stream Lifecycle](#31-stream-lifecycle)  
  - [3.2 Binding](#32-binding)  
  - [3.3 Data-Plane Negotiation](#33-data-plane-negotiation)  
  - [3.4 Gesture Boundaries](#34-gesture-boundaries)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Lifecycle Events](#41-lifecycle-events)  
  - [4.2 Binding Events](#42-binding-events)  
  - [4.3 Data-Plane Negotiation Events](#43-data-plane-negotiation-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Parameters and Automation](#7-relationship-to-parameters-and-automation)

---

## 1. Overview

Gesture streams provide a dedicated path for dynamic control of parameter
values. They allow Aura to:

- send very rapid changes with minimal overhead,
- override automation during active gestures (touch/latch/write),
- feed the audio engine's live cohort with realtime data,
- support hardware surfaces with sub-millisecond response.

Pulse coordinates behaviour, but Signal is the primary realtime consumer.

Gesture streams are **session-scoped** — they do not persist into project
files and are re-established at runtime as needed.

**Control Surface Integration:** Control-surface inputs that are high-resolution or continuous can participate in gesture/value-stream sessions. Pulse may distinguish between high-res *gesture streams* (e.g. knob sweeps, fader movements) and normal control events (e.g. simple button presses). Hardware controllers discovered by Signal can generate gesture streams when mapped to parameters via Pulse's mapping engine.

---

## 2. Gesture Model

### 2.1 Control-Plane vs Data-Plane

#### **Control-plane (Aura ↔ Pulse)**
Via JSON IPC messages:

- create/destroy gesture streams,
- bind gesture streams to `parameterId`s,
- define update-rate hints,
- track gesture boundaries (`beginGesture` / `endGesture`),
- report final values after a gesture,
- integrate with automation semantics (touch/latch/write modes),
- manage permissions, UI representation and cohort invalidation.

Pulse remains the authority for:

- what parameters exist,
- which gesture streams control which parameters,
- how gestures interact with automation and cohorts,
- what values are recorded.

#### **Data-plane (Aura → Signal)**

A high-rate, low-latency channel (WebRTC, named pipe, UNIX socket, shared
memory segment, custom binary channel) carries:

- raw gesture samples,
- at control-surface or UI-driven rates (30–1000Hz).

This channel is:

- **direct** between Aura → Signal,
- **binary**, **non-JSON**, **loss-tolerant**,
- optionally timestamped,
- negotiated and authorised by Pulse,
- capable of being multiplexed between streams.

Pulse does *not* need to see every sample. It only needs *enough* information
to:

- update model state,
- resolve automation,
- drive UI refresh,
- confirm gesture boundaries.

This dual-path model is fundamental and non-negotiable.

---

### 2.2 Streams and Targets

A gesture stream:

- has a unique `gestureStreamId`,
- references one or more `parameterId`s (commonly a single parameter),
- may have scaling/normalisation rules,
- may target multiple parameters (e.g. XY pads, 2D controls, multi-tap nodes),
- always has a single data-plane transport path.

Pulse validates target compatibility with parameter type, range and units.

---

### 2.3 Sample Format and Reliability

Gesture samples over the data-plane are:

- **loss-tolerant**  
  Dropping samples is acceptable; the newest value is authoritative.

- **ordered per stream**  
  Consumers should process samples in-order when timestamps or sequence
  numbers are present.

- **rate-hinted**  
  Aura should respect `maxUpdatesPerSecond` or telemetry provided by Pulse or
  Signal.

- **parameter-type aware**  
  Values must match the target parameter's expected type and range.

Pulse may request decimation for:

- UI updates,
- automation write resolution,
- cohort invalidation events.

Signal may apply smoothing (internal DSP behaviour) but must respect the
parameter's semantics.

---

## 3. Commands (Aura → Pulse)

### 3.1 Stream Lifecycle

#### createStream (command — domain: gesture)

Request creation of a gesture stream.

Fields:

- `description` (optional),  
- `updateRateHint` (optional),  
- `source` metadata (device, control ID, etc.).

Pulse returns:

- `gestureStreamId`.

Pulse then emits a negotiation event for establishing the data-plane channel.

---

#### destroyStream (command — domain: gesture)

Destroy the stream.

Fields:

- `gestureStreamId`.

Pulse invalidates binding, instructs Signal to close the associated data-plane
channel, and emits relevant events.

---

### 3.2 Binding

#### bindParameter (command — domain: gesture)

Attach a gesture stream to one or more parameters.

Fields:

- `gestureStreamId`,
- `parameterId`,
- optional scaling options.

#### unbindParameter (command — domain: gesture)

Detach a stream from its targets.

---

### 3.3 Data-Plane Negotiation

#### negotiateDataChannel (command — domain: gesture)

Optional explicit negotiation call. Establishes or renegotiates the
data-plane transport.

Fields:

- `gestureStreamId`,
- `protocol` (e.g. `"binary"`, `"shared-mem"`, `"pipe"`, `"webrtc"`),
- optional parameters for buffer size, MTU, etc.

Pulse responds with configuration details or a rejection if unsupported.

In most cases, negotiation will be initiated *by Pulse* after
`createStream`, but the IPC contract supports Aura-driven negotiation
where required.

---

### 3.4 Gesture Boundaries

Pulse needs clean gesture boundaries to:

- resolve automation precedence,
- commit final values,
- perform cohort transitions,
- apply parameter smoothing or handover logic.

#### beginGesture (command — domain: gesture)

Fields:

- `gestureStreamId`,
- optional gesture-type metadata (mouse/touch/knob/etc.).

#### endGesture (command — domain: gesture)

Fields:

- `gestureStreamId`,
- optional final value (for redundant sync).

Pulse uses these boundaries to:

- determine write/touch/latch automation behaviour,
- request invalidation of anticipative renders,
- broadcast parameter state updates.

---

## 4. Events (Pulse → Aura)

### 4.1 Lifecycle Events

#### streamCreated (event — domain: gesture)

Stream acknowledged and ready for negotiation.

#### streamDestroyed (event — domain: gesture)

---

### 4.2 Binding Events

#### parameterBound (event — domain: gesture)

#### parameterUnbound (event — domain: gesture)

Pulse alerts Aura when bindings change so UI surfaces and hardware mappings can
stay in sync.

---

### 4.3 Data-Plane Negotiation Events

#### dataChannelReady (event — domain: gesture)

Indicates Pulse and Signal have agreed on a data-plane transport.

Fields include:

- endpoint details (path/URI/shm handle/port/etc.),
- protocol,
- any required authentication tokens.

#### dataChannelFailed (event — domain: gesture)  
Indicates that the data-plane channel cannot be established.

Aura may choose to fall back to Pulse-mediated updates.

---

## 5. Snapshot Semantics

Gesture streams are **not persisted** in project snapshots.

They are session-level constructs.

However:

- persistent “hardware control mapping” rules (e.g. knob 1 always controls
  FaderNode.gain on track X) may be stored elsewhere and used to re-establish
  gesture streams automatically on project load.

Pulse does not encode gesture activity in snapshots.

---

## 6. Realtime and Cohort Considerations

Gesture streams interact directly with:

- automation modes (read/write/touch/latch),
- cohort assignment (live vs anticipative),
- parameter smoothing,
- anticipative render invalidation.

Key rules:

- Signal must receive gesture data with **minimal latency**.
- Pulse must be notified of gesture boundaries and final values.
- When an active gesture targets parameters in anticipative cohorts, Pulse may:
  - promote affected nodes to the live cohort,
  - pause or invalidate anticipative buffers,
  - schedule re-render operations.

Automation precedence:

- In **read** mode: automation wins unless gesture begins.
- In **touch** mode: gesture overrides automation only while active.
- In **latch** mode: gesture overrides automation and retains control after release.
- In **write** mode: gesture edits create/modify automation points.

Gesture streams are the mechanism through which these semantics operate.

---

## 7. Relationship to Parameters and Automation

Gesture domain connects the other control systems:

- **Parameter domain**  
  Gesture streams target `parameterId`s. Pulse resolves type/range and applies
  updates to model state.

- **Automation domain**  
  Gesture boundaries and values drive automation write behaviour.

- **Node domain**  
  Gesture-driven parameter updates propagate to FaderNodes, PanNodes,
  SendNodes, PluginNodes, etc.

- **Mixing model**  
  All mixer fader, pan, send and macro controls use gesture streams under the hood.

Gesture streams are the **foundation** for Loophole’s realtime interaction
model, making precise and responsive control possible even within an
anticipative processing architecture.
