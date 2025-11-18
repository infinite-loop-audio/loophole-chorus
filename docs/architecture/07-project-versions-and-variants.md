# Project, Versions & Variants Architecture

This document defines the **project-level model** for Loophole:

- what a *project* is,
- how **arrangements**, **versions**, and **variants** relate,
- how project state is stored, autosaved, and recovered,
- how Pulse owns and exposes this model to Aura and Signal,
- how this ties into undo/redo and background tasks.

It builds on the high-level architecture in `01-overview.md` and the Pulse
architecture in `02-pulse.md`, and constrains how the `pulse.project` IPC
domain operates.

---

## 1. Goals & Non-Goals

### 1.1 Goals

- Provide a clear, extensible model for:
  - **Projects** (top-level unit of work),
  - **Arrangements** (timeline-specific structures),
  - **Versions** (project- or arrangement-level branches),
  - **Variants** (lightweight alternatives of a given thing).
- Support:
  - safe, explicit saving (`project.save`) separate from background drafting,
  - robust recovery after crashes or device removal,
  - experimentation without duplicating entire projects,
  - future integration with **Composer** for structural analysis.
- Ensure the model is:
  - **Pulse-owned** and deterministic,
  - serialisable and versionable,
  - consistent with Pulse IPC and undo architecture.

### 1.2 Non-Goals

This document does **not**:

- Define the UI for project selection or arrangement switching (Aura concern).
- Define multi-user collaboration semantics (see future Collaboration doc).
- Specify file formats in detail (only the logical storage model).
- Replace the Undo & History architecture (that has its own document); here we
  only define how undo relates to projects/versions at a high level.

---

## 2. Core Concepts

At the top level, the hierarchy is:

- **Project**
  - **Arrangements** (one or more)
    - **Tracks, Lanes, Clips, Nodes, Automation**, etc.
  - **Project-level Metadata** (title, tempo defaults, tags, artwork, etc.)
  - **Project Versions & Variants**

### 2.1 Project

A **project** is the root model entity owned by Pulse and represented on disk
(usually as a single project file plus media references).

Properties (conceptually):

- `projectId` – stable identifier assigned by Pulse.
- `name` – human-readable name.
- `rootPath` – base path for media and project file.
- `arrangements[]` – ordered list of arrangements.
- `versions[]` – project-level version records (see below).
- `meta` – metadata block (see project metadata spec).
- `timebase` – reference to tempo/timebase map used as default.

There is always exactly **one active project** per running Pulse instance
(although Pulse may later support background loading of another on a worker).

### 2.2 Arrangement

An **arrangement** is a specific timeline configuration of tracks, clips,
automation, and nodes.

- Each project has at least one arrangement:
  - `arrangementId`
  - `name` (e.g. “Full Mix”, “Radio Edit”, “Instrumental”)
  - `tracks[]`, `channels[]`, `lanes[]`, `clips[]`, `nodes[]`, etc.
- Arrangements share:
  - **media pool**,
  - **plugins and presets inventory**,
  - but have their own timeline-level structures.

Arrangements are the natural unit for:

- alternate song structures,
- different deliverable mixes,
- experimental “what if” layouts.

### 2.3 Version

A **version** is a *branch* of project state used for more substantial divergences.

There are two levels of versioning:

1. **Project Versions**  
   - Represent the state of the *whole project* at a higher level:
     - arrangement set,
     - global tracks, tempo map, meta,
     - routing configuration.
   - Useful for:
     - “v1 pre-master”, “v2 full re-record”, “v3 rearranged”,
     - long-running projects that evolve over months/years.

2. **Arrangement Versions**  
   - Optional, more fine-grained versions *within* an arrangement:
     - different routing, node graph tweaks,
     - radical automation or track layout changes.
   - Essentially: “branch the arrangement” while keeping project context.

Pulse treats versions as **immutable snapshots plus a pointer**:

- Each version has:
  - `versionId`,
  - `parentVersionId` (optional),
  - `kind` = `"project"` or `"arrangement"`,
  - `createdAt`, `label`, `notes`,
  - a reference to the data needed to reconstruct that version:
    - either embedded in the project file,
    - or stored in a version store (future: for VCS-like behaviour).
- The current, editable state is a mutable working tree that can be:
  - *committed* into a new version,
  - *reverted* to a previous version.

Undo/redo operates *within* the current working tree; versions represent
structural checkpoints above that.

### 2.4 Variant

A **variant** is a *lightweight alternative* of some sub-structure, designed to
support experimentation without full branching overhead.

Examples:

- A **Track Variant**:
  - Different Node graph and automation for a track, but sharing clips.
- A **Clip Variant**:
  - Different internal edits (audio warp points, MIDI tweaks) for the same
    conceptual clip position.
- A **Mix Variant**:
  - Different mixer states (levels, sends, node params) while preserving the
    core arrangement.

Variants are local and scoped:

- identified by `variantId` attached to a parent entity:
  - `trackVariantId` linked to `trackId`,
  - `clipVariantId` linked to `clipId`, etc.
- bounded by the **arrangement** they live in (variants do not cross projects).

Pulse must make variants cheap to create and switch between at the model level
(e.g. by using copy-on-write or structural sharing in its internal model).

---

## 3. Project Lifecycle & State Machine

### 3.1 High-Level States

From Pulse’s perspective, a project has these runtime states:

- `unloaded` – no project in memory.
- `loading` – project is being parsed / hydrated.
- `loaded` – project model is in memory and editable.
- `savingDraft` – background draft/save operation in progress.
- `saving` – explicit `project.save` in progress.
- `closing` – project is being finalised/unloaded.

Only `loaded` state allows editing operations (tracks/lanes/clips/etc.).

### 3.2 Draft vs Save (Two-Step Persistence Model)

Loophole uses a **two-step persistence model**:

1. **Draft Saves** (`project.saveDraft`):
   - Automatic or manual:
     - auto-triggered periodically by Pulse,
     - or triggered when significant changes occur,
     - or requested explicitly by Aura.
   - Writes to a *draft* representation:
     - e.g. `project.loophole.draft` alongside `project.loophole`.
   - Contains the *full current working state*:
     - unsaved edits,
     - pending arrangement changes,
     - possibly partially-completed operations.

2. **Explicit Saves** (`project.save`):
   - Only occur on explicit user intent.
   - Commit the current working state to the **main project file**.
   - May also:
     - create or update a **version record**,
     - prune obsolete draft files.

Recovery logic:

- On startup, if Pulse finds a draft newer than the main project file:
  - it can:
    - offer to restore the draft,
    - or keep both and let the user choose.
- Drafts **never** silently override the main project file.

### 3.3 Project Opening & Snapshot

When opening:

1. Pulse receives `project.open` (Pulse IPC).
2. Pulse loads and parses the project file into an internal model.
3. Pulse emits:
   - `project.loaded` (structural info),
   - followed by `project.snapshot` carrying the *current* model as Pulse sees it.
4. Aura binds its UI state to that snapshot.

The same `project.snapshot` mechanism can be used for:

- switching between versions,
- resetting to a previous state (e.g. after version revert),
- refreshing Aura after background operations (e.g. media re-scan).

---

## 4. Versions & Variants in Detail

### 4.1 Project Versions

A **project version** is a labelled checkpoint of the *whole project*:

- Project versions are created via:
  - explicit user action (e.g. “Create project version…”),
  - or automatically at explicit `project.save` boundaries if configured.

Each version contains:

- reference to:
  - arrangement set,
  - key project-level routing,
  - tempo map,
  - global metadata,
- optional summary:
  - `label` (e.g. “Pre master V1”),
  - `notes` (free text),
  - `tags` (for Composer / future).

Pulse may persist versions:

- inline inside the project file (for simple use),
- OR externalised in a version store (future) for VCS-like functionality.

Version control actions at the model level:

- `createProjectVersion(baseVersionId?)`
- `revertProjectToVersion(versionId)`
- `duplicateProjectVersion(versionId)` (new branch).

### 4.2 Arrangement Versions

Arrangement versions are **arrangement-scoped**:

- Track/routing/clip/automation state for that arrangement only.
- Project-level settings (e.g. global tempo map, metadata) are not versioned
  here; those remain at project-level.

Operations:

- `createArrangementVersion(arrangementId, baseVersionId?)`
- `revertArrangementToVersion(arrangementId, versionId)`
- `duplicateArrangementVersion(arrangementId, versionId)`

These operations must:

- be reflected in the project model,
- propagate to Aura via updated `project.snapshot`,
- be undoable (once undo architecture is fully specified).

### 4.3 Variants

Variants are narrower in scope than versions:

- **Track Variants**:
  - Variation in node graph, automation, and maybe track-level options.
- **Clip Variants**:
  - Different internal warp markers, trimming, or MIDI edits.

Pulse should expose an abstract variant model:

- Each parent entity (track, clip, lane, etc.) can carry:
  - `variants[]` – list of variant descriptors,
  - `activeVariantId` – which one is currently selected.
- Each variant descriptor references:
  - the data needed to reconstruct that variant’s content,
  - possibly shared substructure with other variants (for efficiency).

Variants are not fully-fledged branches of the whole project; they are:

- local,
- cheap to create,
- easy to switch.

Aura can provide UI to:

- create a new variant,
- duplicate an existing one,
- switch active variants per entity,
- convert a variant into a higher-level version (e.g. promote track variant
  into a new arrangement version).

---

## 5. Storage & Serialisation Model

### 5.1 Single Project File vs Project Folder

Pulse should support at minimum:

- A single project file (`.loophole` or similar) that contains:
  - the project model,
  - project and arrangement version metadata,
  - some local configuration (e.g. last used arrangement).
- A project *folder* structure:
  - project file,
  - draft file,
  - caches (e.g. waveform overviews),
  - optional version store,
  - logs/diagnostics for that project.

The **logical model**:

- The project file is the canonical source of truth for:
  - project model,
  - versions metadata,
  - high-level references to media (via the media architecture).

Draft files reflect the current working state that is **not yet committed**.

### 5.2 Backwards-Compatible Evolutions

Given this doc sits early in the architecture set, the project format must:

- support schema evolution:
  - new fields,
  - new version/variant metadata,
  - new domain blocks,
- allow safe loading of older projects:
  - unknown fields are ignored where possible,
  - conversions recorded (for Composer and diagnostics).

Pulse is responsible for:

- project schema versioning,
- migration steps on load,
- emitting warnings where semantics may change.

---

## 6. Interaction with Undo & History

The **Undo & History Architecture** (later document) will define:

- how undo stacks are scoped,
- how branching undo is represented.

From this document’s perspective:

- Undo works *within* the current project working state.
- Version creation operates at a higher level:
  - creating a version may optionally compact/segment undo history.
- Version revert:
  - is treated as a large structural edit that is itself undoable,
  - after revert, undo stack applies within the reverted state.

Conceptually:

- **Undo:** “step back one or a few actions”.
- **Version:** “reset the *overall* state to a known checkpoint while keeping
  future branching possible”.

Pulse must ensure the project model handles both cleanly.

---

## 7. Interaction with Pulse & Signal

### 7.1 Pulse

Pulse is the **authority** for:

- loading, saving, and drafting projects,
- maintaining the in-memory project graph,
- applying edits, versions, and variants,
- exposing snapshots to Aura and commands to Signal.

The `pulse.project` IPC domain must:

- provide commands for `open`, `close`, `save`, `saveDraft`,
- expose events for:
  - `project.loaded`,
  - `project.snapshot`,
  - `project.metaUpdated`,
  - version/variant updates once defined at IPC level.

### 7.2 Signal

Signal is **not aware** of projects explicitly.

It sees:

- graph snapshots,
- media references,
- plugin and routing configuration.

Pulse translates:

- project and arrangement state → engine graph state,
- version/variant decisions → graph mutations or resets.

Signal does not store project or version metadata and should remain agnostic.

---

## 8. Composer Integration (Future Hooks)

Although Composer is not directly responsible for project storage, the project
model should leave room for:

- storing analysis metadata:
  - e.g. “this arrangement is a variation of that one”,
  - project-level tags inferred from structure (dense automation, tempo usage).
- capturing user behaviour:
  - which versions get kept vs discarded,
  - which variants get promoted.

None of this is required for v1, but the project schema must be open enough to
accommodate extra metadata blocks without breaking.

---

## 9. Summary

This document defines:

- **Project** as the root Pulse-owned model,
- **Arrangements** as timeline-specific views,
- **Versions** as structural checkpoints at project and arrangement levels,
- **Variants** as lightweight alternatives attached to entities,
- a **two-step save model** (`saveDraft` vs `save`) for safety,
- the core lifecycle of project loading and snapshotting,
- the boundaries between project storage (Pulse) and graph execution (Signal).

Subsequent architecture documents (Timebase, Tracks & Lanes, Clips, Media,
Automation, Undo & History) must align with these definitions and treat the
project model as the authoritative, evolving container for their own domain
state.
