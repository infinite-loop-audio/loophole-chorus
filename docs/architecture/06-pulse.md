# Pulse Architecture

Pulse is the authoritative data and sequencing layer of the Loophole system. It
hosts the full project model, resolves editing operations, constructs and
maintains engine graphs for Signal, interprets parameter and automation mappings,
coordinates with Composer for enhanced metadata, and provides structured,
incremental updates to Aura and Signal.

Pulse is the “brain” of Loophole: the single, canonical source of truth for all
non-real-time state.

---

## Contents

- [1. Overview](#1-overview)
- [2. Responsibilities](#2-responsibilities)
- [3. Project Model](#3-project-model)
  - [3.1 Data Integrity](#31-data-integrity)
  - [3.2 Snapshot Generation](#32-snapshot-generation)
- [4. Editing Model](#4-editing-model)
  - [4.1 Editing Transactions](#41-editing-transactions)
  - [4.2 Undo/Redo](#42-undoredo)
  - [4.3 Conflict and Overlap Rules](#43-conflict-and-overlap-rules)
- [5. Channels and Processor Graphs](#5-channels-and-processor-graphs)
  - [5.1 Channel Lifecycle](#51-channel-lifecycle)
  - [5.2 Processor Chain Construction](#52-processor-chain-construction)
  - [5.3 LaneStream Nodes](#53-lanestream-nodes)
  - [5.4 Graph Stability Constraints](#54-graph-stability-constraints)
- [6. Parameter Model](#6-parameter-model)
  - [6.1 Parameter Identity](#61-parameter-identity)
  - [6.2 Parameter Storage](#62-parameter-storage)
  - [6.3 Gesture Handling](#63-gesture-handling)
  - [6.4 Automation Resolution](#64-automation-resolution)
- [7. Interaction with Composer](#7-interaction-with-composer)
  - [7.1 Role of Composer](#71-role-of-composer)
  - [7.2 Metadata Acquisition](#72-metadata-acquisition)
  - [7.3 Local Caching](#73-local-caching)
  - [7.4 Advisory Behaviour](#74-advisory-behaviour)
- [8. Interaction with Signal](#8-interaction-with-signal)
  - [8.1 Structural Messages](#81-structural-messages)
  - [8.2 Parameter and Automation Streams](#82-parameter-and-automation-streams)
  - [8.3 High-Rate Gesture Stream](#83-highrate-gesture-stream)
  - [8.4 Engine Graph Rebasing](#84-engine-graph-rebasing)
- [9. Interaction with Aura](#9-interaction-with-aura)
  - [9.1 State Projection](#91-state-projection)
  - [9.2 Validation and Feedback](#92-validation-and-feedback)
  - [9.3 Editing APIs](#93-editing-apis)
- [10. Persistence](#10-persistence)
  - [10.1 Primary Project File](#101-primary-project-file)
  - [10.2 Draft/Autosave Storage](#102-draftautosave-storage)
  - [10.3 Deterministic Serialisation](#103-deterministic-serialisation)
- [11. Future Extensions](#11-future-extensions)

---

## 1. Overview

Pulse is the Loophole project engine. It:

- hosts the project state,
- owns Tracks, Clips, Channels and Lanes,
- resolves editing and performs validation,
- constructs and updates the audio graph for Signal,
- handles all parameter mapping, identity and automation,
- interfaces optionally with Composer for enhanced metadata,
- exposes a stable and predictable API to Aura.

Signal is stateless in terms of project structure; Pulse sends it a resolved and
incremental engine graph. Aura holds no authoritative state and reflects Pulse’s
model.

Pulse may initially run inside Aura as a bundled library but is designed for
process separation without architectural changes.

---

## 2. Responsibilities

Pulse is responsible for:

1. **Project state authority**
   The full project model lives in Pulse only.

2. **Editing and validation**
   Applying edit operations, merging, resolving conflicts, enforcing rules.

3. **Channel and processor graph construction**
   Producing the real-time-safe representation that Signal executes.

4. **Parameter identity, aliasing, mapping and automation**
   Ensuring stable parameter references across plugin changes.

5. **Coordination with Composer**
   Retrieving enhanced plugin/parameter metadata.

6. **Incremental state change emission**
   Sending deltas to Aura and Signal.

7. **Persistence**
   Saving and loading project files, draft state and auxiliary metadata.

---

## 3. Project Model

The project model consists of:

- Tracks and nested Tracks,
- Clips and Lanes,
- Channels and processor chains,
- Parameters, automation, routing,
- Global Lanes (tempo, groove, etc.),
- Media references.

### 3.1 Data Integrity

Pulse enforces invariants such as:

- Clip time ranges must not be inverted,
- Lanes must be valid for Clip type and location,
- Processor chain must remain stable and linear,
- Routing must resolve to valid Channel targets,
- Parameter identities must remain globally unique.

Invalid edits are rejected with structured error responses.

### 3.2 Snapshot Generation

Whenever the project changes, Pulse emits a **snapshot event** to Aura containing
the new UI-relevant projection of model state. Snapshot events do not contain the
entire project; instead they contain appropriately grouped, view-ready model
segments.

Snapshots define the truth for Aura; incremental deltas follow for efficiency.

---

## 4. Editing Model

### 4.1 Editing Transactions

Every editing operation is a transaction:

- atomic changes,
- validation before commit,
- rollback on failure,
- event emission on success.

Transactions may cascade — complex edits (e.g. Track duplication) may consist of
sub-operations stitched into a single commit.

### 4.2 Undo/Redo

Pulse maintains a reversible history of editing operations, not raw state
snapshots. Undo/redo is deterministic and based on inverse operations.

### 4.3 Conflict and Overlap Rules

Pulse defines deterministic conflict resolution rules for:

- overlapping Clips,
- lane routing collisions,
- parameter mapping conflicts,
- processor reordering,
- nested Track inheritance.

Aura is not permitted to invent or assume rules; Pulse defines them.

---

## 5. Channels and Processor Graphs

### 5.1 Channel Lifecycle

A Track may have zero or one Channel. Pulse creates Channels lazily, on demand,
when Clips require audio or instrument processing.

### 5.2 Processor Chain Construction

Pulse constructs the processor chain:

- serial DSP nodes,
- instrument nodes,
- LaneStream nodes,
- sends/returns (future),
- channel gain, pan and metering nodes.

Pulse ensures the chain is:

- stable,
- real-time safe,
- free of cycles or branching,
- suitable for deterministic execution in Signal.

### 5.3 LaneStream Nodes

LaneStreams represent audio from Lanes of Clips. Pulse:

- creates them on first use,
- routes audio Lanes to them,
- allows user-defined splitting (additional LaneStreams),
- inserts them at the top of the processor chain by default.

### 5.4 Graph Stability Constraints

Pulse guarantees:

- no mid-playback structural changes,
- no implicit processor insertion or removal,
- stable ordering unless explicitly edited,
- deterministic processor IDs.

Signal receives graph updates only when necessary, and always in compact,
incremental form.

---

## 6. Parameter Model

### 6.1 Parameter Identity

Pulse maintains canonical IDs for all parameters:

- track/channel parameters,
- processor parameters,
- automation targets,
- global parameters.

IDs remain stable across duplication, routing changes and plugin updates.

### 6.2 Parameter Storage

Pulse holds:

- current parameter value,
- default value,
- scaling/normalisation rules,
- associated metadata (range, units),
- mapping tables for aliases.

### 6.3 Gesture Handling

Aura sends high-rate parameter gesture streams. Pulse:

- updates parameter state,
- emits real-time gesture stream to Signal,
- stores commit-point values if user confirms,
- avoids polluting automation lanes unless explicitly requested.

### 6.4 Automation Resolution

Automation Lanes target fully qualified parameter IDs. When resolving automation
for playback:

- Pulse evaluates automation curves,
- merges gesture streams if applicable,
- clamps values to parameter range,
- emits high-resolution automation samples to Signal.

---

## 7. Interaction with Composer

### 7.1 Role of Composer

Composer is a metadata service external to the core engine. Pulse may consult
Composer for:

- plugin identity normalisation,
- parameter metadata (names, ranges, units),
- semantic roles,
- mapping suggestions,
- version/format equivalence.

Composer is **never required** for project correctness.

### 7.2 Metadata Acquisition

Pulse fetches Composer metadata:

- lazily on first need,
- cached locally,
- invalidated per plugin variant.

### 7.3 Local Caching

Pulse stores Composer metadata in a project-local cache so that:

- the project loads perfectly offline,
- metadata persists across sessions,
- deterministic behaviour is maintained.

### 7.4 Advisory Behaviour

Composer suggestions are advisory. Pulse always:

- verifies mapping correctness,
- resolves conflicts deterministically,
- maintains full authority.

Signal has zero awareness of Composer.

---

## 8. Interaction with Signal

### 8.1 Structural Messages

Pulse sends Signal:

- Channel creation/destruction,
- processor chain updates,
- routing changes,
- parameter registration.

Only minimal diffs are sent; Pulse owns the full model.

### 8.2 Parameter and Automation Streams

Pulse emits:

- normal parameter updates,
- automation updates,
- modulation (future),
- value smoothing instructions.

Signal receives a normalised, real-time-safe representation.

### 8.3 High-Rate Gesture Stream

For real-time control, Pulse forwards gesture streams directly to Signal via a
dedicated connection. These do not modify automation unless committed.

### 8.4 Engine Graph Rebasing

When major structural changes occur, Pulse may send a rebased graph to Signal.
Rebasing is:

- infrequent,
- atomic,
- explicitly triggered by major edits.

---

## 9. Interaction with Aura

### 9.1 State Projection

Pulse emits UI-ready projections (“snapshots”) that describe:

- tracks,
- clips,
- lanes,
- automation,
- processors,
- parameters,
- routing.

Aura must never derive structure independently.

### 9.2 Validation and Feedback

Aura sends editing intents. Pulse validates them and returns either:

- success + updated snapshot, or
- structured error describing the issue.

### 9.3 Editing APIs

Pulse provides:

- high-level editing commands (e.g., “split clip”, “duplicate track”),
- lower-level operations (parameter/set, routing/set, etc.),
- transactionally safe modifications.

Aura never mutates the model directly.

---

## 10. Persistence

### 10.1 Primary Project File

Pulse serialises:

- tracks,
- clips and lanes,
- routing,
- parameters,
- processor chains,
- media references,
- Composer metadata cache,
- alias tables.

### 10.2 Draft/Autosave Storage

Pulse maintains:

- draft project state updated regularly,
- commit-point state only when user performs a save.

### 10.3 Deterministic Serialisation

Saved projects must:

- be deterministic,
- avoid ordering ambiguity,
- not rewrite unchanged data.

---

## 11. Future Extensions

Pulse is designed for long-term extensibility. Possible future additions include:

- modulation system,
- graph-based Clip-dependent routing,
- multi-engine bridging,
- distributed DSP topologies,
- multi-project linked editing,
- dynamic device discovery.
