# Pulse Transport Domain Specification

This document defines the Transport domain of the Pulse Project Protocol.
It covers the commands issued by Aura to Pulse and the events emitted by Pulse
in response to transport control actions.

Transport commands represent **user intent** (play, stop, seek, loop) at the
project/model level. Pulse interprets these commands, updates its internal
transport state and issues appropriate engine commands to Signal via the Engine
Integration Protocol.

High-resolution timing and visual playhead updates are provided separately via
the telemetry plane from Signal to Aura and do not pass through this domain.

---

## Contents

- [1. Overview](#1-overview)
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)
  - [2.1 Basic Transport Control](#21-basic-transport-control)
  - [2.2 Looping](#22-looping)
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)
  - [3.1 State Changes](#31-state-changes)
  - [3.2 Errors](#32-errors)
- [4. Interaction with Signal and Telemetry](#4-interaction-with-signal-and-telemetry)

---

## 1. Overview

Transport behaviour in Loophole is defined at the model level by Pulse:

- Aura expresses transport intent via commands (play, stop, seek, loop changes).
- Pulse updates its internal transport state and sends engine commands to Signal.
- Signal attempts to realise the requested state (start/stop playback, etc.).
- Pulse reconciles engine responses and emits transport events back to Aura.

Pulse is responsible for the **semantic meaning** of transport operations, while
Signal is responsible for real-time execution.

### Routing Roles

The routing model for transport operations is:

- **Aura → Pulse**: sends user transport intents (`transport.play`,
  `transport.pause`, `transport.seek`, etc.).
- **Pulse → Signal**: sends derived transport commands with full timing context.
- **Signal → Pulse**: may send confirmations or engine-status related events but
  never initiates transport changes.

---

## 2. Commands (Aura → Pulse)

### 2.1 Basic Transport Control

**`transport.play`**
Begin playback from the current transport position, or from a specified position
if the command payload includes one.

Pulse:

- updates its transport state to `playing` (subject to engine confirmation),
- instructs Signal to start playback from the chosen position.

**`transport.stop`**
Stop playback and seek to a specified position. This is distinct from
`transport.stop`:

- if a target position is provided in the payload, Pulse sets the transport
  position accordingly after stopping;
- if no position is provided, Pulse may use a default policy (for example,
  seeking to the start of the project or to the last play-start position).

Pulse:

- updates its transport state to `stopped`,
- adjusts its internal transport position,
- instructs Signal to stop playback and seek.

**`transport.seek`**
Move the transport position without changing play/stop state.

- If called while stopped, this is a simple reposition of the playhead.
- If called while playing, the semantics depend on Pulse’s rules (for example,
  jump to a new playback position).

The seek target (time or musical position) is carried in the command payload.

---

### 2.2 Looping

**`transport.setLoopRegion`**
Set the loop region boundaries.
Payload includes:

- loop start position,
- loop end position,

in the chosen coordinate system (time, musical or both).

**`transport.setLoopEnabled`**
Enable or disable looping.

Pulse:

- updates its transport state to reflect the new loop configuration,
- issues appropriate engine commands to Signal to apply loop behaviour.

---

## 3. Events (Pulse → Aura)

### 3.1 State Changes

**`transport.stateChanged`**
Emitted whenever the semantic transport state changes. This includes:

- transitions between playing, stopped (and paused, if introduced later),
- updates to Pulse’s notion of the transport position,
- changes to loop enablement,
- changes to loop region.

Typical payload fields include:

- `state`: `playing` | `stopped` | `paused` (if supported),
- `position`: current transport position (coarse model-level view),
- `loopEnabled`: boolean,
- `loopRegion`: start and end positions.

Aura should rely on `transport.stateChanged` to update transport controls and
high-level UI elements (e.g. play/stop/loop button states). Fine-grained
playhead position for drawing is provided by telemetry from Signal.

---

### 3.2 Errors

**`transport.error`**
Emitted when a transport command cannot be fulfilled. Examples include:

- no active or valid audio device,
- Signal refusing to start playback,
- project not in a valid state for the requested command.

Typical payload fields include:

- `command`: the originating transport command (e.g. `transport.play`),
- `code`: machine-readable error code,
- `message`: human-readable description,
- `details`: optional structured context (for logging or diagnostics).

Aura may surface these errors to the user, log them or both.

---

## 4. Interaction with Signal and Telemetry

The Transport domain concerns the model-level representation of playback:

- Pulse receives commands from Aura and updates its internal transport state.
- Pulse issues engine commands to Signal in the Engine Integration Protocol
  (e.g. engine-level play, stop, seek, loop configuration).
- Signal reports engine outcomes back to Pulse, which reconciles them before
  emitting `transport.stateChanged` and any relevant `transport.error` events.

High-resolution transport information (e.g. continuous playhead position and
timing data) is delivered directly from Signal to Aura via the telemetry plane,
and is not part of the Pulse Project Protocol.

### Cohort and Anticipative Implications

When transport seeks, loops, or jumps:

- Pulse may instruct Signal to **invalidate anticipative buffers** and rebuild
  the render horizon.
- Transport operations are still defined by the existing IPC commands; the new
  engine architecture changes **how Signal internally responds**, not the IPC
  shape.

These side effects are handled internally by Signal's dual-engine architecture
(Anticipative and Live engines) without requiring changes to the transport
command schemas.
