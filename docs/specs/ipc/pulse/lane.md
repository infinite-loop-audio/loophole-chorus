# Pulse Lane Domain Specification

This document defines the Lane domain of the Pulse Project Protocol.  
It covers the commands issued by Aura to Pulse to create, remove or modify
Lanes, and the events emitted by Pulse to notify Aura of Lane-level changes.

A Lane represents a **media-bearing or behaviour-bearing stream** within a Track
or Clip. Each Lane:

- has a stable `laneId`,
- has a `laneType` (MIDI, audio, automation, video, device, etc.),
- may appear in one or more Clips,
- may carry content (handled by the corresponding Lane-type domain),
- may be routed to a LaneStreamNode (often shortened to 'LaneStream') in the Track's Channel,
- appears in a Clip in a user-defined order for editing.

Pulse owns all Lane state. Aura expresses editing intent only.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Lane Types](#2-lane-types)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Creation and Removal](#31-creation-and-removal)  
  - [3.2 Identity and Presentation](#32-identity-and-presentation)  
  - [3.3 Routing and LaneStream Association](#33-routing-and-lanestream-association)  
  - [3.4 Structural Attachment](#34-structural-attachment)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Structural Events](#41-structural-events)  
  - [4.2 Identity and Presentation Events](#42-identity-and-presentation-events)  
  - [4.3 Routing Events](#43-routing-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)

---

## 1. Overview

A Lane is a typed content container used by Clips to store musical, audio,
visual or automation data. Tracks may provide default Lanes, but Clips may
introduce additional Lanes as needed (e.g. layered audio or multi-instrument
MIDI).

Lanes:

- belong logically to a Track,
- are referenced by Clips,
- may or may not carry content at a given Clip position,
- may route to one or more LaneStreamNode(s) (often shortened to 'LaneStream') in the Track's Channel,
- may specify presentation properties (colour, name, collapsed state).

The Lane domain governs the existence, identity, presentation and routing of
Lanes. Content editing (e.g. MIDI notes, audio regions, automation curves)
belongs to the Lane-type-specific domains defined elsewhere.

---

## 2. Lane Types

Pulse recognises core Lane types:

- **`midi`** – note data, CCs, channel pressure, etc.
- **`audio`** – references to audio regions and sample ranges.
- **`automation`** – parameter automation envelopes.
- **`video`** – video regions or timeline-referenced frames.
- **`device`** – device interaction/event data for hardware or control surfaces.

Additional types may be defined in future domain expansions.

### Automation Lane Workflow (Informative)

The Automation domain is responsible for expressing *what* should be automated
and *how* points are edited. The Lane domain is responsible for providing the
timeline container (an automation lane) that lives alongside other Lanes on a
Track.

A typical automation lane creation flow is:

1. Aura decides that a parameter should be automated (for example, the user
   invokes "Show Automation" on a Channel fader or plugin parameter).

2. Aura (or the Automation domain) determines the target `parameterId` and the
   owning context (Track, Lane, Channel, Node).

3. Pulse ensures that a corresponding automation lane exists on the relevant
   Track/Lane by:

   - creating a new automation lane entry in the Lane model if none exists,

   - associating it with the target `parameterId`.

4. After the automation lane exists, Automation domain commands and events
   (adding/updating/removing points) operate on that lane's data.

5. Aura renders the automation lane in the editor using the Lane identity and
   Automation domain's point data.

The Lane domain does **not** define automation point editing itself; it only
owns:

- the existence and identity of automation lanes,

- their placement relative to other Lanes on a Track,

- any visual or ordering metadata Aura requires.

Automation data (points, shapes, curves) always lives in the Automation domain.

---

## 3. Commands (Aura → Pulse)

Aura issues Lane commands in response to user intent. Pulse applies the
commands, updates the project model and emits events.

### 3.1 Creation and Removal

**`lane.create`**  
Create a new Lane on a Track.

Payload fields include:

- `trackId`,
- `laneType`,
- optional presentation metadata (colour, name),
- optional initial routing hints.

Pulse allocates a stable `laneId`.

**`lane.delete`**  
Remove an existing Lane. Pulse ensures:

- the Lane is detached from all Clips,
- routing references are resolved,
- dependent content (MIDI, audio, automation) is handled by the Lane-type domain.

---

### 3.2 Identity and Presentation

**`lane.rename`**  
Change the Lane’s user-visible name.

**`lane.setColour`**  
Assign or clear a Lane-specific colour. This is presentation metadata; Pulse
stores it and Aura interprets it visually.

**`lane.setCollapsed`**  
Set whether a Lane should appear collapsed in editing views.

Payload fields include:

- `laneId`,
- `collapsed` (boolean).

---

### 3.3 Routing and LaneStream Association

Tracks may have multiple LaneStreamNodes in their Channel. Lanes specify
which LaneStreamNode receives their output.

**`lane.setLaneStream`**  
Assign the Lane's output to a specific LaneStreamNode.

Fields:

- `laneId`,
- `laneStreamId` (nullable to unset routing). This `laneStreamId` refers to the `nodeId` of a LaneStreamNode in the Channel's node graph. See the Node domain specification for LaneStreamNode details.

Pulse updates routing and may issue Channel/Node domain updates.

**`lane.setSendLevel`**  
Set the mix level or weighting when a Lane outputs to a LaneStreamNode.

This applies primarily to audio or video Lanes.

---

### 3.4 Structural Attachment

Lanes may be attached to or detached from Clips.

**`lane.attachToClip`**  
Attach a Lane to a Clip. Payload:

- `laneId`,
- `clipId`.

**`lane.detachFromClip`**  
Detach a Lane from a Clip. Pulse ensures content visibility rules are applied.

**`lane.reorderInClip`**  
Reorder Lanes within a Clip’s editor-facing lane list.

Payload:

- `clipId`,
- `laneIds` (ordered list).

---

## 4. Events (Pulse → Aura)

Pulse emits Lane events after valid command application or when Lane state
changes due to higher-level operations (e.g. Clip or Track snapshot loads).

### 4.1 Structural Events

**`lane.created`**  
A new Lane has been created. Includes:

- `laneId`,
- `trackId`,
- `laneType`,
- initial presentation metadata.

**`lane.deleted`**  
Indicates a Lane has been removed.

**`lane.attachedToClip`**  
Lane now appears within the Clip.

**`lane.detachedFromClip`**  
Lane removed from the Clip’s lane list.

**`lane.reorderedInClip`**  
Presentation ordering changed for a Clip.

---

### 4.2 Identity and Presentation Events

**`lane.renamed`**  
The Lane’s name has been updated.

**`lane.colourChanged`**  
Colour metadata updated.

**`lane.collapsedChanged`**  
Collapsed/expanded state updated.

---

### 4.3 Routing Events

**`lane.laneStreamChanged`**  
The Lane's associated LaneStreamNode has changed.

**`lane.sendLevelChanged`**  
Mixing or routing weight updated.

Pulse may dispatch additional Channel or Node domain events if the Lane's
routing change requires Channel graph updates.

---

## 5. Snapshot Semantics

Lane information is included in project-level snapshots, grouped under their
own section or as part of Track/Clip snapshot data.

Lane snapshot entries include:

- identity (`laneId`),
- `trackId`,
- `laneType`,
- presentation metadata (`name`, `colour`, `collapsed`),
- routing (`laneStreamId`, send levels),
- list of Clips to which the Lane is attached (ordered),
- any additional static metadata required by the Lane-type domain.

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

---

## 6. Realtime and Cohort Considerations

Lane operations are non-realtime. They may cause:

- Channel graph changes (e.g. creation or removal of LaneStreamNodes),
- reassignment of routing paths,
- impacts on cohort selection if routing affects determinism assumptions.

No Lane domain operation occurs on the audio thread.  
Pulse ensures routing and Channel changes are committed safely before issuing
Pulse → Signal updates in other domains.
