# Pulse Parameter Domain Specification

This document defines the Parameter domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse to manipulate parameter values
and control groups, and the events emitted by Pulse to notify Aura of parameter
state changes.

Parameters represent the primary control surface for:

- Node behaviour (plugin parameters, built-in DSP controls),
- Channel behaviour (fader gain, pan, meter modes),
- Track-level behaviour (track-wide options or macros),
- Control Groups (VCA-style relative control),
- selected project-level options.

Pulse is the authoritative owner of parameter identity, types, and value
resolution. Aura expresses user intent, visualises state, and may attach UI
controls to parameters.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Parameter Identity and Types](#2-parameter-identity-and-types)  
- [3. Parameter Ownership](#3-parameter-ownership)  
- [4. Commands (Aura → Pulse)](#4-commands-aura--pulse)  
  - [4.1 Direct Value Control](#41-direct-value-control)  
  - [4.2 Gesture-Based Control](#42-gesture-based-control)  
  - [4.3 Control Groups (VCA-style Behaviour)](#43-control-groups-vca-style-behaviour)  
  - [4.4 Metadata and Presentation](#44-metadata-and-presentation)  
- [5. Events (Pulse → Aura)](#5-events-pulse--aura)  
  - [5.1 Value and State Events](#51-value-and-state-events)  
  - [5.2 Gesture Events](#52-gesture-events)  
  - [5.3 Control Group Events](#53-control-group-events)  
- [6. Snapshot Semantics](#6-snapshot-semantics)  
- [7. Realtime and Cohort Considerations](#7-realtime-and-cohort-considerations)  
- [8. Relationship to Automation and Gesture Streams](#8-relationship-to-automation-and-gesture-streams)

---

## 1. Overview

The Parameter domain provides a unified model for all controllable values in
Loophole, including:

- plugin parameters,
- built-in DSP node controls (e.g. FaderNode gain, PanNode position),
- Channel-level controls (fader, pan, metering configuration),
- Track-level controls (where appropriate),
- Control Group “master” controls (VCA-style behaviour).

Key responsibilities of this domain:

- define stable parameter identity,
- define parameter types, ranges and units,
- receive and resolve value changes from Aura,
- apply relative transformations via control groups,
- emit events when effective parameter values change.

Automation (time-based parameter modulation) is handled by the Automation
domain; the Parameter domain defines the *target* values and control inputs
automation operates on.

For the broader conceptual model, see
[Parameters Architecture](../../architecture/08-parameters.md) and
[Mixing Model and Console Architecture](../../architecture/11-mixing-console.md).

---

## 2. Parameter Identity and Types

Each parameter has a stable `parameterId`.  
The `parameterId` is:

- globally unique within a project,
- stable across saves and minor edits,
- derived from a combination of:

  - owner context (Node, Channel, Track, or Control Group),
  - local identifier within that context (e.g. plugin parameter ID),
  - versioning rules to support plugin upgrades and mapping (with Composer).

The Parameter domain assumes a strict type system. Core types include:

- `float` – continuous scalar (e.g. gain in dB, 0–1 normalised controls),
- `integer` – discrete steps (e.g. number of voices),
- `enum` – one-of-many options (e.g. filter mode),
- `boolean` – on/off toggles.

Each parameter also has:

- a nominal range (e.g. min, max, step),
- a display unit (e.g. dB, %, Hz, semitones),
- a normalisation scheme for UI controls (0–1 or suitable mapping).

Parameter type and metadata are defined by the owning entity (Node, Channel,
Track) and may be extended by Composer for cross-plugin mapping scenarios.

---

## 3. Parameter Ownership

Parameters are owned by higher-level entities:

- **Node-owned parameters**  
  PluginNode parameters, LaneStreamNode configuration, SendNode level/pan,
  FaderNode gain, PanNode position, AnalyzerNode modes, etc.

- **Channel-owned parameters**  
  Channel fader/pan may be modelled as Node-owned parameters; any remaining
  Channel-level configuration that behaves like a parameter is represented here.

- **Track-owned parameters**  
  Track-wide, non-DSP parameters or macros that do not belong to a specific
  Node but still require control and automation.

- **Control Group parameters**  
  Parameters representing group-level controls that apply relative changes to a
  set of target parameters (VCA-style).

The Parameter domain makes no assumption about the internal structure of Nodes
or Channels; it only requires a consistent mapping between owners and their
parameters.

---

## 4. Commands (Aura → Pulse)

Aura issues Parameter commands when:

- the user adjusts a control in the UI,
- a control group is edited,
- metadata/presentation hints are updated.

Pulse validates, resolves precedence (e.g. gesture vs automation), updates the
project model, and emits events.

### 4.1 Direct Value Control

**`parameter.setValue`**  
Set the value of a parameter directly.

Payload fields include:

- `parameterId`,
- `value` (according to the parameter’s type).

This represents an immediate, non-gesture change (e.g. UI toggle, jump).

Pulse applies the change, resolves any group interactions and automation
precedence, and emits `parameter.valueChanged` if the effective value changes.

---

### 4.2 Gesture-Based Control

Gesture-based control supports continuous adjustments (e.g. dragging a fader).
It allows Aura to mark a gesture window to apply smoothing, automation
capture and precedence correctly.

**`parameter.beginGesture`**  
Indicate the start of a user gesture for a parameter.

Fields:

- `parameterId`,
- optional metadata such as:

  - source (e.g. mouse, MIDI controller, touch),
  - gestureId (optional correlation token).

**`parameter.updateGesture`**  
Provide a new value for a parameter during an active gesture.

Fields:

- `parameterId`,
- `value`.

**`parameter.endGesture`**  
Indicate the end of a user gesture. Pulse may use this boundary for automation
writes or other gesture-related logic.

Fields:

- `parameterId`.

Aura should always send `beginGesture` and `endGesture` for continuous
operations where meaningful. For simple clicks or toggles, Aura may send only
`parameter.setValue`.

---

### 4.3 Control Groups (VCA-style Behaviour)

Control Groups are defined in the Parameter domain to implement VCA-style
behaviour and other grouped control patterns. They do not pass audio.

**`parameter.createGroup`**  
Create a new Control Group.

Fields include:

- group name,
- optional role (e.g. `fader`, `sendLevel`, `macro`).

Pulse returns/announces a `groupId`, which may also be represented as a
`parameterId` for the group’s master control.

**`parameter.deleteGroup`**  
Delete an existing Control Group. Pulse removes group associations and cleans
up any related master controls.

**`parameter.setGroupMembers`**  
Define or update the set of member parameters in a Control Group.

Fields:

- `groupId`,
- list of member descriptors (each referencing a `parameterId` and optional
  weighting or mode).

Pulse uses this to apply relative changes from the group’s master parameter to
its members.

**`parameter.setGroupMasterValue`**  
Set the value of a group master parameter (can also use `parameter.setValue`
if the group master is represented as a normal parameter). Pulse calculates
relative adjustments to member parameters based on the group configuration.

---

### 4.4 Metadata and Presentation

**`parameter.setPresentation`**  
Set presentation metadata for a parameter. This may include:

- display name or alias,
- groupings or categories (e.g. “Filter”, “Envelope”),
- preferred UI control style hints (e.g. knob, slider).

Pulse stores these hints; Aura interprets them when rendering control surfaces.
Renderer-specific choices remain Aura’s responsibility.

---

## 5. Events (Pulse → Aura)

Pulse emits Parameter events after applying valid commands or when parameter
values change as a consequence of automation, routing, or other internal
updates.

### 5.1 Value and State Events

**`parameter.valueChanged`**  
Indicates that the effective value of a parameter has changed.

Includes:

- `parameterId`,
- new value,
- optional metadata about the source of change (e.g. automation, gesture,
  group).

Aura uses this to update displayed controls.

**`parameter.published`**  
Optionally used when a parameter becomes available or its metadata changes,
including:

- identity,
- type and range,
- ownership information (Node/Channel/Track),
- presentation hints.

This allows Aura to discover parameters dynamically.

---

### 5.2 Gesture Events

**`parameter.gestureBegan`**  
Optional event indicating that a gesture has been recognised and accepted
internally. May be used to confirm initiation of a write or to reflect
external gesture sources.

**`parameter.gestureEnded`**  
Optional event indicating the end of a gesture as recognised by Pulse.

Aura may not need to rely on these events if it already tracks gestures
locally, but they can be useful in distributed setups.

---

### 5.3 Control Group Events

**`parameter.groupCreated`**  
Indicates that a Control Group has been created. Includes:

- `groupId`,
- name,
- role.

**`parameter.groupDeleted`**  
Indicates that a Control Group has been removed.

**`parameter.groupMembersChanged`**  
Indicates the member list or weightings have changed for a group.

Aura may use these events to build and update Control Group UIs that mimic or
extend traditional VCA workflows.

---

## 6. Snapshot Semantics

Project snapshots include full parameter state.  
Snapshot data may include:

- all parameter identities and metadata,
- current effective values (including results of automation at the snapshot
  reference point, if applicable),
- Control Group definitions and membership,
- presentation hints where they are considered part of the project state.

Automation curves and envelopes are handled by the Automation domain, which
references the same `parameterId`s.

On snapshot application, Pulse replaces parameter state according to the
snapshot definition and emits `parameter.valueChanged` and/or
`parameter.published` as needed for Aura to synchronise UI.

---

## 7. Realtime and Cohort Considerations

Parameter changes affect Nodes that are assigned to cohorts (Live vs
Anticipative). Pulse is responsible for:

- dispatching parameter changes to Signal threads in a realtime-safe manner,
- integrating gesture updates with cohort-managed nodes,
- invalidating anticipative buffers when parameter changes require re-render.

The Parameter domain itself is not directly executed on the audio thread, but
its commands and events must be compatible with realtime-safe propagation.

Cohort semantics are defined in the Processing Cohorts architecture document.

---

## 8. Relationship to Automation and Gesture Streams

The Parameter domain interacts closely with:

- **Automation domain**  
  Automation references `parameterId`s and schedules time-based changes.
  Automation and direct parameter changes may both target the same parameter;
  Pulse is responsible for resolving precedence rules (e.g. “automation writes
  unless user is actively gesturing”).

- **Gesture and meter streams**  
  For high-rate data (e.g. continuous controller input from hardware), Aura
  and/or Signal may use dedicated gesture streams. The Parameter domain
  defines the *target parameters* that those streams ultimately update.

The exact precedence policy (e.g. automation vs gesture, latch vs touch modes)
is defined in the Automation semantics and gesture handling documents. The
Parameter domain provides a stable, typed target space for those controls.
