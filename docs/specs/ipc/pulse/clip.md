# Pulse Clip Domain Specification

This document defines the Clip domain of the Pulse Project Protocol.
It covers the commands issued by Aura to Pulse to create, manipulate and remove
Clips, and the events emitted by Pulse to notify Aura of changes.

A Clip is a timeline-region container attached to a Track. A Clip may:

- contain one or more Lanes (MIDI, audio, automation, video, device),
- reference material in those Lanes,
- define temporal properties (start, length, offsets, looping),
- define content transforms (timebase, quantisation, slip, stretch),
- exist within a Track’s sequence of Clips (Track domain defines the ordering).

Pulse is the **authoritative owner** of Clip state, including identity,
structural updates, lane references and temporal behaviour.  
Aura expresses editing intent only.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)  
  - [2.1 Creation and Removal](#21-creation-and-removal)  
  - [2.2 Position and Duration](#22-position-and-duration)  
  - [2.3 Looping, Stretching and Slip](#23-looping-stretching-and-slip)  
  - [2.4 Lane Management](#24-lane-management)  
  - [2.5 Content Editing](#25-content-editing)  
  - [2.6 Duplication](#26-duplication)  
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)  
  - [3.1 Structural Events](#31-structural-events)  
  - [3.2 Temporal and Content Events](#32-temporal-and-content-events)  
  - [3.3 Lane Events](#33-lane-events)  
- [4. Snapshot Semantics](#4-snapshot-semantics)  
- [5. Realtime and Cohort Considerations](#5-realtime-and-cohort-considerations)

---

## 1. Overview

A Clip in Loophole is a **multi-lane region** on a Track’s timeline. Clips are
not restricted to a single media type; instead, they may contain a set of Lanes
(MIDI, audio, automation, video, device) with data applying to the region the
Clip occupies.

Each Clip has:

- a stable `clipId`,
- a `trackId` indicating ownership,
- a start position and length in musical or absolute time,
- optional looping behaviour,
- per-lane content managed in the Lane domain.

Clip operations may trigger updates to:

- Lane routing,
- Track-level summaries,
- Channel lane-stream assignments,
- timeline-level playback interpretations.

Clips are not real-time objects; they exist purely in the project model and
affect playback via Pulse → Signal graph and content interpretation.

---

## 2. Commands (Aura → Pulse)

Aura issues Clip commands in response to user actions. Pulse validates and
applies them, updating the project model and emitting events.

### 2.1 Creation and Removal

**`clip.create`**  
Create a Clip within a Track.

Payload fields include:

- `trackId`,
- `start` (timeline position),
- `length`,
- optional initial lane list (if omitted, empty Clip is created),
- optional flags such as initial loop state.

Pulse assigns a new `clipId` and emits `created` (event — domain: clip).

**`clip.delete`**  
Remove a Clip entirely. Pulse validates and updates the Track’s Clip list.

---

### 2.2 Position and Duration

**`clip.move`**  
Move a Clip to a new position or Track.

Fields include:

- `clipId`,
- `newTrackId` (optional; omission means same Track),
- `newStart`.

Pulse maintains correct ordering on both source and destination Tracks.

**`clip.resize`**  
Change the Clip’s duration.

Fields:

- `clipId`,
- `newLength`.

Pulse ensures integrity of lane content (e.g., trimming or revealing regions).

**`clip.split`**  
Split a Clip at a given timeline position.  
Pulse creates two new Clips (or updates the existing and creates one new).

Fields:

- `clipId`,
- `position` (split point).

**`clip.merge`**  
Merge two adjacent Clips, if compatible.  
Pulse creates a new Clip or extends one existing Clip based on content rules.

---

### 2.3 Looping, Stretching and Slip

**`clip.setLooping`**  
Enable or disable looping. Includes:

- `loopEnabled`,
- optional `loopLength` (if explicitly defined).

**`clip.setTimeStretch`**  
Set Clip-level stretching behaviour.

- can be musical (BPM-relative) or absolute,
- may affect audio, MIDI and other Lanes.

**`clip.slip`**  
Move the content inside the Clip without changing the Clip’s outer bounds.

Fields:

- `clipId`,
- `delta` (time shift applied to all lane content).

Pulse applies lane-relative offsets internally.

---

### 2.4 Lane Management

Clips reference Lanes (defined in the Lane domain). Clip commands may manage
those references.

**`clip.addLane`**  
Attach a Lane to the Clip.

Fields:

- `clipId`,
- `laneId`.

**`clip.removeLane`**  
Detach a Lane from the Clip. Pulse ensures that removed lane content becomes
invisible or must be removed in the Lane domain.

**`clip.reorderLanes`**  
Define presentation ordering of Lanes within a Clip.

Fields:

- `clipId`,
- `laneIds` (ordered list).

(This is purely structural; content is handled by the Lane domain.)

---

### 2.5 Content Editing

Clip-level commands do not operate directly on audio/MIDI/automation data.
That content is managed by the Lane domain. Clip domain supports container-level
modifications.

**`clip.setContentBounds`**  
Adjust internal content bounds relative to Clip boundaries.

Fields:

- `clipId`,
- `startOffset`,
- `endOffset`.

**`clip.quantise`**  
Quantise all quantisable lane content within a Clip to a given grid.

Fields may include:

- grid type,
- intensity,
- whether to include audio transient markers.

Quantisation of audio, MIDI or automation is delegated to Lane-specific
implementations.

---

### 2.6 Duplication

**`clip.duplicate`**  
Duplicate a Clip or group of Clips.

Payload:

- `clipId` (source),
- `newTrackId` (optional),
- `newStart`.

Pulse replicates the Clip, its lane attachments and associated content (Lane
domain handles cloning content internally).

---

## 3. Events (Pulse → Aura)

Pulse emits Clip events following successful mutation or when Clips change as
a consequence of snapshot application.

### 3.1 Structural Events

**`created`** (event — domain: clip)  
A new Clip has been created. Includes:

- `clipId`,
- `trackId`,
- `start`, `length`,
- initial Lane list (laneIds),
- loop state.

**`deleted`** (event — domain: clip)  
Clip removed from the project. Aura must drop related UI state.

**`moved`** (event — domain: clip)  
Clip moved to a new Track and/or position.

Includes:

- `clipId`,
- old and new `trackId` (if applicable),
- old and new `start`.

**`resized`** (event — domain: clip)  
Clip length changed.

---

### 3.2 Temporal and Content Events

**`loopingChanged`** (event — domain: clip)  
Loop enable/disable or parameters updated.

**`stretchChanged`** (event — domain: clip)  
A Clip's stretching rules have changed.

**`slipped`** (event — domain: clip)  
Indicates content-offset shifts.

---

### 3.3 Lane Events

**`laneAdded`** (event — domain: clip)  
A Lane has been attached to a Clip.

**`laneRemoved`** (event — domain: clip)  
A Lane has been detached from a Clip.

**`lanesReordered`** (event — domain: clip)  
Lane order changed.

---

## 4. Snapshot Semantics

Clip state appears in project-level snapshots emitted by Pulse (e.g.
`snapshot` — kind: snapshot, domain: project). Clip snapshot entries may include:

- identity (`clipId`),
- track association,
- start and length,
- looping parameters,
- stretching parameters,
- slip offset,
- list of attached `laneId`s (ordered),
- content bounds (if any).

Lane and content-specific snapshots exist in the Lane domain.

### Snapshot Application

When Aura receives a `snapshot` (kind: snapshot — domain: project) that includes this domain, it MUST treat
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

## 5. Realtime and Cohort Considerations

Clip operations are non-realtime and must not run on Signal’s audio thread.

However, Clip changes may have implications for:

- Lane routing (which may alter Channel graph structure),
- required nodes (e.g. LaneStreamNode(s), often shortened to 'LaneStream'),
- cohort assignment if Lane content implies routing changes.

Any necessary graph updates or cohort reassignment are handled by Pulse
via other domains.
