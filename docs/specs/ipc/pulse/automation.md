# Pulse Automation Domain Specification

This document defines the Automation domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse for creating, editing and
managing automation data, and the events emitted by Pulse to notify Aura of
automation changes.

Automation represents **time-based control** of parameters, attached to Tracks,
Lanes and Clips via automation lanes. Automation drives parameter values during
playback, in cooperation with the Parameter domain and gesture handling.

Pulse is the authoritative owner of automation envelopes and their attachment
to parameters. Aura expresses editing intent and visualises envelopes in the
UI.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Automation Concepts](#2-automation-concepts)  
  - [2.1 Targets and Parameter Binding](#21-targets-and-parameter-binding)  
  - [2.2 Scopes: Track-, Lane- and Clip-Relative Automation](#22-scopes-track--lane--and-clip-relative-automation)  
  - [2.3 Envelopes and Points](#23-envelopes-and-points)  
  - [2.4 Automation Modes (Read / Off / Write / Touch / Latch)](#24-automation-modes-read--off--write--touch--latch)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Envelope Lifecycle](#31-envelope-lifecycle)  
  - [3.2 Point Editing](#32-point-editing)  
  - [3.3 Range Editing](#33-range-editing)  
  - [3.4 Mode and Write-Arming](#34-mode-and-write-arming)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Envelope Events](#41-envelope-events)  
  - [4.2 Point Events](#42-point-events)  
  - [4.3 Mode and Write-State Events](#43-mode-and-write-state-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

Automation in Loophole provides time-based control of parameters such as:

- plugin parameters on PluginNodes,
- FaderNode gain and PanNode position,
- SendNode levels and pans,
- LaneStreamNode configuration,
- Track- or project-level macro parameters.

Automation is represented as **envelopes** attached to automation lanes.  
An envelope:

- targets a specific `parameterId` (Parameter domain),
- exists in a particular scope (Track/Lane/Clip),
- contains a sequence of time-stamped points with interpolation.

Pulse:

- stores and evaluates automation,
- resolves precedence between automation, direct parameter changes and gestures,
- coordinates with processing cohorts (PC) and anticipative rendering.

For conceptual background see:

- [Parameters Architecture](../../architecture/10-parameters.md)  
- [Mixer & Channel Architecture](../../architecture/12-mixer-and-channel-architecture.md)

---

## 2. Automation Concepts

### 2.1 Targets and Parameter Binding

Automation always targets a parameter. The binding is expressed via:

- a `parameterId` (from the Parameter domain),
- optionally additional context for display (e.g. owning Node/Channel/Track).

Automation lanes are therefore **parameter-specific**: one envelope per lane
per parameter. Multiple envelopes for the same parameter may exist in different
scopes (e.g. Clip-level vs Track-level).

Pulse is responsible for:

- ensuring `parameterId` is valid,
- resolving any conflicts when multiple scopes affect the same parameter.

### 2.2 Scopes: Track-, Lane- and Clip-Relative Automation

Automation may exist in several scopes, for example:

- **Track-level automation**  
  Automation envelope attached directly to a Track (e.g. overall fader).

- **Lane-level automation**  
  Automation envelope attached to a particular Lane (e.g. per-lane pan).

- **Clip-level automation**  
  Automation envelope whose timebase is relative to a Clip’s start and moves
  with the Clip when edited.

The Automation domain assumes that the existence of automation lanes and their
attachment to Tracks/Lanes/Clips is managed by the Lane and Clip domains.  
Automation commands operate on **envelopes associated with existing automation
lanes**, identified by an `automationId` (or equivalent lane identifier) that
Pulse provides.

### 2.3 Envelopes and Points

An envelope is modelled as:

- a timebase (musical or absolute time; typically project timeline),
- a collection of points, each with:

  - time position,
  - value,
  - interpolation shape (e.g. `step`, `linear`, `curve`, `bezier`),
  - optional `shapeParams` (e.g. an array of numbers).

Interpolation behaviour between points is defined by a combination of:

- point-level shape metadata (including `shape` and optional `shapeParams`),
- optional envelope-level default interpolation.

The `shape` field indicates the interpolation type (e.g. `step`, `linear`,
`curve`, `bezier`). The optional `shapeParams` field may carry additional
parameters that influence the curve shape, such as curve tension or normalised
Bezier handle positions. The exact interpretation of `shapeParams` is an
internal Pulse concern and may evolve; the IPC schema supports rich vector-style
curve shapes from the outset.

The Automation domain does not mandate a specific interpolation set or curve
algorithm; Pulse may support a limited set initially and extend it later.

### 2.4 Automation Modes (Read / Off / Write / Touch / Latch)

Automation interacts with live parameter control via a set of modes, including:

- **Read** – automation drives the parameter where envelopes exist.
- **Off** – automation is ignored; parameters follow direct/gesture control.
- **Write** – parameter changes overwrite or create automation data while
  transport is running.
- **Touch** – automation is written while a gesture is active; when released,
  automation reverts to existing values.
- **Latch** – automation continues to write the last value after gesture
  release until the mode changes or transport stops.

Modes may be defined:

- per parameter,
- per Track/Channel (default for parameters on that Channel),
- or per global session.

This specification defines IPC-level commands and events; precise precedence
policies are documented in a dedicated automation semantics document.

---

## 3. Commands (Aura → Pulse)

Aura issues automation commands when:

- the user edits envelopes directly (drawing, moving, deleting points),
- the user changes automation modes,
- the user arms parameters or tracks for writing.

Pulse validates commands, updates envelope data, and emits events reflecting
changes.

### 3.1 Envelope Lifecycle

**`automation.createEnvelope`**  
Create an automation envelope for a parameter in a specific scope.

Fields include:

- `target` – identifier for the scope, e.g.:

  - Track-level: `trackId`,
  - Lane-level: `laneId`,
  - Clip-level: `clipId` plus possibly `laneId`.

- `parameterId`,
- optional initial points,
- optional interpolation defaults.

Pulse returns/announces an `automationId` representing this envelope.

> Note: the creation of the underlying automation lane (as a Lane with
> `laneType = "automation"`) is handled by the Lane domain. This command
> binds an envelope to that lane.

**`automation.deleteEnvelope`**  
Delete an automation envelope (and its points).

Fields:

- `automationId`.

Pulse removes the envelope and any associated automation points.

---

### 3.2 Point Editing

Automation points are the primary edit units.

**`automation.addPoint`**  
Add a point to an envelope.

Fields include:

- `automationId`,
- `time` (envelope-relative time),
- `value`,
- optional `shape` (interpolation type),
- optional `shapeParams`.

**`automation.updatePoint`**  
Update an existing point (e.g. during drag).

Fields:

- `automationId`,
- `pointId`,
- new `time`, `value`, and/or `shape` (and optionally `shapeParams`).

**`automation.setPointShape`**  
Change the interpolation shape (and optional shape parameters) of a point
without changing its `time` or `value`.

Fields:

- `automationId`,
- `pointId`,
- `shape`,
- optional `shapeParams`.

This command is separate from `automation.updatePoint`, which is primarily
intended for changing a point’s time or value (though it may also touch shape if
desired).

**`automation.deletePoint`**  
Delete an individual point.

Fields:

- `automationId`,
- `pointId`.

Pulse may re-index points internally; `pointId` is an opaque identifier from
Aura’s perspective.

---

### 3.3 Range Editing

Range operations support more complex editing gestures.

**`automation.clearRange`**  
Clear all automation points in a time range.

Fields:

- `automationId`,
- `timeStart`,
- `timeEnd`.

**`automation.moveRange`**  
Move a selection of points in time.

Fields:

- `automationId`,
- `timeStart`,
- `timeEnd`,
- `deltaTime`.

**`automation.scaleRange`**  
Scale values in a time range (e.g. trim automation up/down or compress
dynamics).

Fields:

- `automationId`,
- `timeStart`,
- `timeEnd`,
- `scale` (e.g. multiplicative factor or other defined scheme).

**`automation.curveRange`**  
Request that Pulse reshapes automation in a time range using a curve descriptor
(e.g. a simple “bend” or “tilt” control), with Pulse free to adjust underlying
points as needed.

Fields:

- `automationId`,
- `timeStart`,
- `timeEnd`,
- `curve` (e.g. a single scalar specifying curvature or other curve descriptor).

This command provides a higher-level operation for bulk curve manipulation,
allowing Aura to request curve transformations without issuing thousands of
explicit point edits. Precise behaviour and curve descriptor formats may be
refined in a dedicated automation editing guidelines document.

Exact semantics of scaling may be refined in a dedicated automation editing
guidelines document.

---

### 3.4 Mode and Write-Arming

**`automation.setMode`**  
Set the automation mode for a target context.

Fields:

- `target` (e.g. `parameterId`, `trackId`, or `channelId`),
- `mode` (e.g. `read`, `off`, `write`, `touch`, `latch`).

Pulse stores mode and uses it during playback and gesture handling.

**`automation.armWrite`**  
Arm a parameter or context for automation writing.

Fields:

- `target` (similar to `setMode`),
- optional `armed` boolean (true/false).

Write-arming determines which parameters will receive automation data when a
gesture occurs in a write-capable mode.

---

## 4. Events (Pulse → Aura)

Pulse emits automation events:

- after commands are applied,
- when automation changes as a side-effect (e.g. snap to grid, consolidation),
- when modes or write-arming statuses change.

### 4.1 Envelope Events

**`automation.envelopeCreated`**  
A new envelope is available.

Includes:

- `automationId`,
- target scope,
- `parameterId`,
- optional initial summary (e.g. bounds).

**`automation.envelopeDeleted`**  
Envelope has been removed.

---

### 4.2 Point Events

**`automation.pointAdded`**  
Point has been added to an envelope.

**`automation.pointUpdated`**  
Point has changed (time/value/shape).

**`automation.pointDeleted`**  
Point has been removed.

**`automation.rangeChanged`**  
Used when Pulse performs bulk operations (e.g. clear, move, scale) and prefers
to send a single event rather than many point-level events. Aura may request
full envelope data if needed.

---

### 4.3 Mode and Write-State Events

**`automation.modeChanged`**  
Automation mode for a target has changed.

**`automation.writeArmedChanged`**  
Write-arming state has changed for a parameter or context.

These events keep Aura’s mode indicators and transport controls in sync with
Pulse’s authoritative automation state.

---

## 5. Snapshot Semantics

Automation is an integral part of project snapshots. Snapshot data includes:

- all envelopes, with:

  - `automationId`,
  - target scope (Track/Lane/Clip),
  - `parameterId`,
  - complete point lists and interpolation data.

- automation modes and write-arming state where considered part of project
  state (session vs project settings may be distinguished in future).

### Snapshot Application

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

Snapshots do not change Parameter identity; they operate over the existing set
of `parameterId`s. Handling of missing or remapped parameters (e.g. due to
plugin changes) is coordinated with Composer and Parameter-domain rules.

---

## 6. Realtime and Cohort Considerations

Automation editing is non-realtime and must not affect the audio thread
directly. However, automation playback is critical to audio behaviour.

Pulse is responsible for:

- ensuring automation evaluation is efficient and cohort-aware,
- propagating automation-driven parameter changes to the correct Node in
  Signal in a realtime-safe way,
- invalidating anticipative renders when automation changes affect the
  anticipative window.

Cohort details and anticipative behaviour are described in the [Processing Cohorts](../architecture/06-processing-cohorts-and-anticipative-rendering.md) architecture document. The Automation domain provides the time-based
control data; cohort logic determines when and how those changes affect
rendered audio.

---

## 7. Relationship to Other Domains

Automation interacts with several domains:

- **Parameter domain**  
  Automation envelopes always target `parameterId`s. The Parameter domain
  defines the type, range and meaning of those parameters, and resolves
  precedence between automation and direct/gesture control.

- **Lane domain**  
  Automation lanes are represented as Lanes with `laneType = "automation"`.
  The Lane domain is responsible for their creation, deletion and attachment
  to Tracks/Clips. The Automation domain operates on envelopes bound to those
  lanes.

- **Clip and Track domains**  
  Clip-level and Track-level automation scopes are defined in the Clip and
  Track models. Automation must follow Clip moves, copies and edits.

- **Routing, Channel and Node domains**  
  Many automation targets are Node-owned parameters (e.g. FaderNode gain,
  SendNode level) that directly influence Channel graphs and routing.

- **Gesture streams**  
  High-rate control data may be delivered via gesture streams. The Automation
  domain defines how such gestures are recorded into envelopes, or how they
  temporarily override automation in touch/latch modes.

Further details of these interactions will be expanded in dedicated semantics
documents (automation behaviour, gesture handling, and cohort scheduling).
