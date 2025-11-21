# Pulse Project Domain Specification

This document defines the Project domain of the Pulse Project Protocol.
It covers the commands issued by Aura to Pulse and the events emitted by
Pulse in response to lifecycle changes, persistence actions, identity updates
and snapshot requests.

The Project domain establishes the foundational behaviour for all project-level
operations and ensures a clear separation between:

- project lifecycle,
- project persistence and commit behaviour,
- project identity and metadata,
- and how Pulse communicates project state back to Aura.

---

## Contents

- [1. Overview](#1-overview)
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)
  - [2.1 Lifecycle](#21-lifecycle)
  - [2.2 Persistence and Commit Behaviour](#22-persistence-and-commit-behaviour)
  - [2.3 Identity and Metadata](#23-identity-and-metadata)
  - [2.4 Inspection](#24-inspection)
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)
  - [3.1 Lifecycle Events](#31-lifecycle-events)
  - [3.2 Persistence Events](#32-persistence-events)
  - [3.3 Identity and Metadata Events](#33-identity-and-metadata-events)
  - [3.4 Snapshot and Meta Events](#34-snapshot-and-meta-events)
- [4. Snapshot Semantics](#4-snapshot-semantics)
- [5. Draft vs Primary Project State](#5-draft-vs-primary-project-state)

---

## 1. Overview

The Project domain governs the lifecycle and storage of a Loophole project.
Aura issues commands representing user intent; Pulse validates the request,
updates internal state and emits events reflecting the outcome.

Pulse is always the **source of truth** for the in-memory project model.

Aura never mutates project data directly and does not read project files.
All project state visible to Aura is carried through `snapshot` events (domain: project) and
related events.

### Routing Model

Pulse is the **only** authority for project state. The routing model for
project-related operations is:

- **Aura → Pulse**: sends project-related commands (open, save, rename, etc.).
- **Pulse → Aura**: sends project snapshots and project-related events.
- **Pulse → Signal**: sends graph rebuild instructions after project load changes.
- **Signal → Pulse**: may send engine-level status/errors but never initiates
  project-level changes.

Signal never directly processes project-level commands. All project state
mutations flow through Pulse.

---

## 2. Commands (Aura → Pulse)

### 2.1 Lifecycle

#### new (command — domain: project)

Create a new empty project in memory. No primary storage location is required.

#### open (command — domain: project)

Open a project from a specified location (path or URI).
Triggers `loaded` events followed by `snapshot` events on success.

On success, `open` triggers:
- model load in Pulse,
- **fresh cohort assignment**,
- a new engine graph build for Signal,
- emission of `snapshot` events toward Aura.

#### close (command — domain: project)

Close the current project and reset Pulse's project state.

---

### 2.2 Persistence and Commit Behaviour

Pulse maintains two conceptual copies of project state:

- a **draft** state, regularly persisted during editing, and
- a **primary** committed project file, updated only when the user explicitly saves.

Commands:

#### saveDraft (command — domain: project)

Persist the current in-memory project state to the draft location.
This may be called explicitly by Aura or invoked on a timer within Pulse.

#### save (command — domain: project)

Commit the current project state to the **primary project file**.
Fails if the project does not yet have a primary location.

#### saveAs (command — domain: project)
Commit to a new primary project location.
On success, the new location becomes the primary project file.

---

### 2.3 Identity and Metadata

#### rename (command — domain: project)
Change the project’s title.

#### move (command — domain: project)
Move the project’s primary file (and optionally associated assets) to a new
location.

#### updateMeta (command — domain: project)
Apply updated metadata such as author, description, notes or tags.

---

### 2.4 Inspection

**`requestMeta`** (command — domain: project)  
Request that Pulse emit the project's current metadata via a `meta` event (kind: event — domain: project).

**`requestSnapshot`** (command — domain: project)  
Request a full project snapshot via `snapshot` (kind: snapshot — domain: project).

---

## 3. Events (Pulse → Aura)

### 3.1 Lifecycle Events

**`loaded`** (event — domain: project)  
Emitted after Pulse has successfully loaded the project into memory following
`project.open` or `project.new`.
This event does **not** contain project data.

Following `loaded`, Pulse performs fresh cohort assignment and prepares
graph rebuild instructions for Signal. The subsequent `snapshot` event (kind: snapshot)
carries the complete project state to Aura.

**`loadFailed`** (event — domain: project)  
The project could not be opened due to file I/O, format or version issues.

**`closed`** (event — domain: project)  
The project was closed and the model has been cleared.

---

### 3.2 Persistence Events

**`draftSaved`** (event — domain: project)  
Draft state successfully persisted.

**`draftSaveFailed`** (event — domain: project)  
Draft save failed.

**`saved`** (event — domain: project)  
Primary project file successfully committed via `project.save` or
`project.saveAs`.

**`saveFailed`** (event — domain: project)  
Primary commit failed.

---

### 3.3 Identity and Metadata Events

**`renamed`** (event — domain: project)  
Emitted after a successful rename.

**`renameFailed`** (event — domain: project)  
Rename was rejected or failed.

**`moved`** (event — domain: project)  
Primary project location successfully moved.

**`moveFailed`** (event — domain: project)  
Move failed.

**`metaUpdated`** (event — domain: project)  
Metadata successfully updated.

**`metaUpdateFailed`** (event — domain: project)  
Metadata update failed validation or persistence.

---

## 3.4 Snapshot and Meta Events

*(Existing text preserved exactly, with the following addition appended at the end of this section.)*

### Snapshot Payload Shape (Initial Minimal Schema)

The `project.snapshot` event carries the **complete authoritative state** of the
project as Pulse currently understands it. The payload is versioned and may grow
over time, but MUST maintain backward-compatible semantics.

The minimal `ProjectSnapshot` payload schema is:

```ts
interface ProjectSnapshot {
  projectId: string;
  path: string | null;
  name: string;

  status: "empty" | "new" | "loaded" | "dirty";

  timebase: {
    tempo: number;
    timeSignature: [number, number];
  };

  summary: {
    trackCount: number;
    channelCount: number;
    hasUnsavedChanges: boolean;
  };

  // Additional optional fields may be added in future
  // without breaking Aura, provided new fields are additive.
}
```

Pulse MUST ensure that the snapshot schema evolves monotonically, maintaining
existing fields and adding new ones without removing or renaming established
keys. Aura MUST treat unknown fields as opaque extensions.

### Emission Rules (Clarified)

- `loaded` (event — domain: project) **never carries data**. It only indicates that a project has
  been successfully opened and is resident in memory.

- `snapshot` (kind: snapshot — domain: project) MUST always follow `loaded` and MUST be the **only**
  source of authoritative project state visible to Aura.

- `snapshot` MUST also be emitted:
  - after `project.new`,
  - after `project.save`, `project.saveAs`, or `project.saveDraft` (if state changed),
  - after `project.updateMeta`,
  - after any internal structural change Pulse deems significant,
  - whenever Aura explicitly requests one via `project.requestSnapshot`.

Snapshots are **replacements**, not merges. Aura MUST discard any outdated local
state for the Project domain when receiving one.

---

## 4. Snapshot Semantics

Pulse emits `snapshot` (kind: snapshot — domain: project) to provide Aura with a complete representation of
the current project. This differs from `loaded` (event — domain: project):

- `loaded` indicates that the project exists in memory,
- `snapshot` conveys the actual project data.

Snapshots may be used for:

- initial UI population,
- UI re-synchronisation,
- versioned project views (future feature),
- recovery after errors.

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

### Domain-Level Scoping (Clarified)

Snapshots apply to the **project domain only**. Other domains (tracks, lanes,
timebase, mixer, routing, etc.) will define their own snapshot formats where
needed. Aura should treat each domain independently and apply snapshot updates
per-domain.

---

## 5. Draft vs Primary Project State

Pulse maintains separate draft and primary states:

- **Draft state** updates frequently and may be persisted via `project.saveDraft`.
  This enables background safety saves and uninterrupted user workflow.

- **Primary state** is only updated via `project.save` or `project.saveAs`.
  This provides predictable user-driven commits.

Aura should treat the draft location as opaque; only Pulse manages draft files.
