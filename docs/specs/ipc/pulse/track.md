# Pulse Track Domain Specification

This document defines the Track domain of the Pulse Project Protocol.
It covers the commands issued by Aura to Pulse and the events emitted by
Pulse in response to creation, deletion, hierarchy changes, identity updates,
processing policy updates and channel–track binding state.

Tracks in Loophole form the structural backbone of the project timeline.
A Track may:

- exist in a hierarchical tree (parent/child relationships),
- own a Channel (DSP processing chain),
- contain Clips and Lanes,
- propagate user-facing flags (mute, solo, arm, monitor),
- define processing policy hints for processing cohort (PC) assignment.

Pulse is always the **source of truth** for Track structure and Track metadata.
Aura never mutates Track data directly.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)  
  - [2.1 Creation and Deletion](#21-creation-and-deletion)  
  - [2.2 Hierarchy and Ordering](#22-hierarchy-and-ordering)  
  - [2.3 Identity and Presentation](#23-identity-and-presentation)  
  - [2.4 Flags and Processing Policy](#24-flags-and-processing-policy)  
  - [2.5 Channel Binding](#25-channel-binding)  
  - [2.6 Duplication](#26-duplication)  
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)  
  - [3.1 Structural Events](#31-structural-events)  
  - [3.2 Identity and Presentation Events](#32-identity-and-presentation-events)  
  - [3.3 Flag and Policy Events](#33-flag-and-policy-events)  
  - [3.4 Channel Binding Events](#34-channel-binding-events)  
- [4. Snapshot Semantics](#4-snapshot-semantics)  
- [5. Realtime and Cohort Considerations](#5-realtime-and-cohort-considerations)

---

## 1. Overview

The Track domain governs all aspects of Track structure and metadata within
a project. Tracks are hierarchical entities that may contain other Tracks,
Clips and Lanes, and may optionally own a Channel for DSP processing.

Track-owned Channels are one class of Channel. Other Channels (busses, sends,
returns, master, I/O, etc.) are defined at the project/routing level and exist
independently of Tracks.

Pulse validates and applies all Track operations, updates the project model,
and emits events notifying Aura of the resulting changes. Pulse may trigger
operations in other domains (Channel, Lane, Node, Routing) when Track
state changes require them.

Tracks are identified by stable `trackId` values. Hierarchy is maintained using
`parentId` and `index` fields.

### Routing Model

All Track domain IPC adheres to the routing rules established by the envelope
specification:

- **Aura → Pulse**: sends Track-related commands (create, delete, move, rename).
- **Pulse → Aura**: sends Track-related events (create, delete, move, rename, etc.) that share the same `name` as their corresponding commands.
- **Pulse → Signal**: may send Channel/graph updates as a consequence of Track
  changes, but this occurs in other domains.
- **Signal → Pulse**: Signal never initiates Track-level behaviour.

---

## 2. Commands (Aura → Pulse)

Aura issues Track commands in response to user intent. Pulse validates and
applies them, then emits the corresponding events.

### 2.1 Creation and Deletion

**`create`** (command — domain: track)  
Create a new Track under a specified parent, at a specified index. If the Track
is intended to host audio/MIDI processing, Aura may request that Pulse attach a
Channel at creation time.

Payload fields include:

- parentId (nullable, for top-level Tracks),
- index (insertion position),
- name,
- initial colour or icon (optional),
- optional Track flags,
- optional processing policy hints,
- createChannel (boolean).

**`delete`** (command — domain: track)  
Delete an existing Track. Pulse validates that deletion is legal (e.g., cannot
remove required system Tracks) before applying the change.

---

### 2.2 Hierarchy and Ordering

**`move`** (command — domain: track)  
Reparent or reorder a Track. Pulse updates:

- the Track's parent,
- the Track's index within the new parent's child list,
- the ordering of sibling Tracks.

This command does not modify Track identity or its Channel.

---

### 2.3 Identity and Presentation

**`rename`** (command — domain: track)  
Rename an existing Track.

**`setColour`** (command — domain: track)  
Assign or clear a Track colour.

**`setIcon`** (command — domain: track)  
Set a Track icon reference. Icons are presentation metadata used by Aura; Pulse
treats them as opaque.

---

### 2.4 Flags and Processing Policy

**`setFlags`** (command — domain: track)  
Update one or more user-facing flags, such as:

- mute,
- solo,
- arm,
- monitor input.

Unspecified flags remain unchanged. Pulse applies the update and emits a
`setFlags` event (kind: event, name: `setFlags`, cid: <command.id>).

**`setProcessingPolicy`** (command — domain: track)  
Set processing policy hints influencing cohort assignment. The `mode` field is:

- `auto` – Pulse decides based on routing and metadata,
- `forceLive` – request live cohort where safe,
- `preferAnticipative` – request anticipative rendering where safe.

Pulse stores this hint and may trigger a cohort re-evaluation.

---

### 2.5 Channel Binding

**`setChannelEnabled`** (command — domain: track)  
Attach or detach a Track-owned Channel from a Track:

- When enabling and no Channel exists, Pulse creates a Track-owned Channel
  and initial LaneStreamNode(s) (often shortened to 'LaneStream').
- When disabling, Pulse removes or deactivates the Track-owned Channel
  according to project rules.

This command only affects Channels owned by this Track. Other Channels
(busses, master, etc.) are managed separately via routing/project-level
commands.

Pulse emits `setChannelEnabled` event (kind: event, name: `setChannelEnabled`, cid: <command.id>).

---

### 2.6 Duplication

**`duplicate`** (command — domain: track)  
Duplicate a Track and optionally include:

- Clips,
- automation,
- Channel,
- Lanes.

Pulse generates new IDs and applies the duplication at the model level.

---

## 3. Events (Pulse → Aura)

Pulse emits Track events after applying valid commands or when internal changes
occur (e.g. as part of snapshot generation).

### 3.1 Structural Events

**`create`** (event — domain: track)  
Emitted after Pulse has successfully processed a `create` command. Indicates a Track has been created.

- Correlates to command: `cid = <create command.id>`
- Payload includes identity, hierarchy, name, colour, flags, processing policy and initial Channel attachment state.

**`delete`** (event — domain: track)  
Emitted after Pulse has successfully processed a `delete` command. Indicates a Track has been removed.

- Correlates to command: `cid = <delete command.id>`
- Aura should drop any associated UI state.

**`move`** (event — domain: track)  
Emitted after Pulse has successfully processed a `move` command. Indicates a Track has been reparented or reordered.

- Correlates to command: `cid = <move command.id>`
- Payload includes old and new parent/index information.

**`rename`** (event — domain: track)  
Emitted after Pulse has successfully processed a `rename` command. Name successfully updated.

- Correlates to command: `cid = <rename command.id>`

**`setColour`** (event — domain: track)  
Emitted after Pulse has successfully processed a `setColour` command. Colour has changed.

- Correlates to command: `cid = <setColour command.id>`

**`setIcon`** (event — domain: track)  
Emitted after Pulse has successfully processed a `setIcon` command. Icon metadata updated.

- Correlates to command: `cid = <setIcon command.id>`

(Pulse may choose to combine these into a single metadata event depending on
internal design, but this specification expresses them separately.)

**`setFlags`** (event — domain: track)  
Emitted after Pulse has successfully processed a `setFlags` command. Mute/solo/arm/monitor flags successfully updated.

- Correlates to command: `cid = <setFlags command.id>`

**`setProcessingPolicy`** (event — domain: track)  
Emitted after Pulse has successfully processed a `setProcessingPolicy` command. Processing policy hint has changed.

- Correlates to command: `cid = <setProcessingPolicy command.id>`
- Aura may visualise this but the meaning is enforced by Pulse and Signal.

**`setChannelEnabled`** (event — domain: track)  
Emitted after Pulse has successfully processed a `setChannelEnabled` command. Indicates when a Track-owned Channel is attached to or detached from a Track.

- Correlates to command: `cid = <setChannelEnabled command.id>`

**`duplicate`** (event — domain: track)  
Emitted after Pulse has successfully processed a `duplicate` command. Indicates a Track has been duplicated.

- Correlates to command: `cid = <duplicate command.id>`

Aura may use this to update mixer visibility or channel strip presentation.
Note that this event only concerns Track-owned Channels; standalone Channels
are managed via routing/project-level events.

---

## 4. Snapshot Semantics

Track state is included in project-level snapshots (see the Project domain).
A snapshot may contain:

- Track identity,
- hierarchy (parentId, index),
- name, colour, icon,
- flags,
- processing policy,
- Channel binding state.

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

The Track domain does not define its own standalone snapshot event.

---

## 5. Realtime and Cohort Considerations

Track domain operations are **never** processed on the audio thread. They may,
however, trigger changes that lead to Signal graph rebuilds or cohort updates
(e.g. changing Track flags or processing policy).

Processing policy hints directly feed into the processing cohort system:
[@chorus:/docs/architecture/a06-processing-cohorts-and-anticipative-rendering.md](../../architecture/a06-processing-cohorts-and-anticipative-rendering.md)

Pulse determines whether Track-level changes require:

- reassignment of nodes between live and anticipative cohorts,
- invalidation of anticipative buffers,
- or further updates in the Channel/Node domains.

Aura is not involved in cohort decisions.
