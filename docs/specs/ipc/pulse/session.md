# Pulse Session Domain Specification

This document defines the Session domain of the Pulse project protocol.  
It covers high-level, human-facing **session and engine state**, separate
from low-level diagnostics.

Where Engine Diagnostics focuses on detailed performance and cohort metrics,
the Session domain answers higher-level questions such as:

- Is the engine ready/running?
- Is the project “dirty” (needs save)?
- Are background tasks active?
- Is the engine in safe-mode?
- Are there global warnings the user should see?

Session state is the canonical “status header” for the DAW.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Session Concepts](#2-session-concepts)  
  - [2.1 Engine Readiness and Mode](#21-engine-readiness-and-mode)  
  - [2.2 Project Dirty State](#22-project-dirty-state)  
  - [2.3 Background Tasks](#23-background-tasks)  
  - [2.4 Safe-Mode and Global Warnings](#24-safe-mode-and-global-warnings)  
  - [2.5 Future Integration: Undo/Redo and Scripting](#25-future-integration-undoredo-and-scripting)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Session Snapshot and Subscriptions](#31-session-snapshot-and-subscriptions)  
  - [3.2 Engine Control Hooks](#32-engine-control-hooks)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Session State Events](#41-session-state-events)  
  - [4.2 Background Task Events](#42-background-task-events)  
  - [4.3 Global Warning Events](#43-global-warning-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Relationship to Diagnostics, Undo and Scripting](#6-relationship-to-diagnostics-undo-and-scripting)

---

## 1. Overview

The Session domain provides a compact, structured view of the current
session’s status:

- engine state (ready/running/suspended),
- project dirty flag,
- background activity (e.g. analysis, renders, media relinking),
- global warnings (e.g. safe-mode, missing plugins, missing media).

It is intended for:

- status bars,
- “session info” panels,
- top-level app state (e.g. whether to enable “Save” vs “Save As”).

Lower-level details (CPU load, cohort contents, plugin crashes, hardware
events) are exposed by other domains.

---

## 2. Session Concepts

### 2.1 Engine Readiness and Mode

Engine state here is a simplified view derived from:

- engine initialisation state,
- hardware/device configuration state,
- core engine run/stop state.

Typical values:

- `uninitialised` – engine not yet set up,
- `initialising` – engine setting up devices/graph,
- `ready` – engine initialised, not currently playing,
- `running` – engine processing audio (play and/or record),
- `suspended` – engine temporarily stopped by an error or explicit action,
- `safeMode` – engine running with reduced features (e.g. problematic plugins
  disabled, anticipative processing off).

Session does not define *how* these modes are entered; it standardises how
they are reported.

---

### 2.2 Project Dirty State

Session tracks whether the project has unsaved changes:

- `clean` – project matches last saved state,
- `dirty` – project has unsaved modifications.

Dirty state is driven by:

- changes to the project model (Tracks/Clips/Nodes/etc.),
- changes to session-level configuration (timebase, hardware roles, etc.),
- undo/redo operations (when implemented).

Session is the single place Aura consults to decide:

- whether to show “unsaved changes” indicators,
- whether to prompt on exit.

---

### 2.3 Background Tasks

Background tasks cover operations such as:

- audio analysis (e.g. waveform overviews, loudness),
- media relinking,
- file conversions,
- offline bounces/renders,
- Composer synchronisation.

Session exposes:

- whether any background tasks are running,
- high-level summaries of those tasks (for a background tasks panel),
- a count or progress indication where appropriate.

The details of each task’s internal behaviour live in the relevant domain
(Media, Bounce/Freeze, Composer, etc.); Session aggregates enough to give
users a sense of “something is happening”.

---

### 2.4 Safe-Mode and Global Warnings

When serious problems occur (e.g. repeated plugin crashes, severe device
issues), the engine may enter a **safe-mode**:

- disabling anticipative processing,
- bypassing or unloading problematic Nodes,
- muting certain Channels.

Session records:

- whether safe-mode is active,
- a short description of why (e.g. “Plugin X crashed repeatedly and has been
  disabled”, “Device Y failed to initialise”).

Global warnings may also be:

- project-wide (e.g. missing media, missing plugins),
- environment-related (e.g. sample rate mismatch with hardware),
- configuration-related (e.g. unsupported combination of devices).

Session carries these as simple, structured alerts; details live in other
domains (Hardware I/O, Engine Diagnostics, Media, Plugin).

---

### 2.5 Future Integration: Undo/Redo and Scripting

The Session domain is the natural place to surface high-level information
about:

- **Undo/redo** (future domain):

  - whether there is undoable/redoable history,
  - a short label for the next undo/redo action (e.g. “Undo: Move Clip”).

- **Scripting/automation** (future domain):

  - whether scripting is enabled/disabled,
  - whether any scripts or automation jobs are currently running.

This specification reserves space for these concepts without defining the
full undo or scripting protocols. When those domains are added, Session will
mirror their high-level state via additional fields/events.

---

## 3. Commands (Aura → Pulse)

### 3.1 Session Snapshot and Subscriptions

#### **`session.requestStatus`**

Request the current Session status snapshot.

Fields: none.

Pulse responds via `session.statusUpdated`.

---

#### **`session.subscribe`**

Subscribe to Session status changes.

Fields (optional):

- `includeBackgroundTasks` (bool),
- `includeWarnings` (bool).

This allows Aura to receive push updates instead of polling.

---

#### **`session.unsubscribe`**

Stop receiving Session status updates.

---

### 3.2 Engine Control Hooks

Strictly speaking, engine start/stop and transport are handled by the
Transport domain. However, Session may provide a **high-level hook** for
“global engine (re)initialisation” when environment changes occur (e.g.
hardware devices lost/restored).

Optional commands:

#### **`session.requestEngineRestart`**

Request a restart of the audio engine, for example after major hardware
changes.

Pulse may:

- refuse (if unsafe at current time),
- queue the restart,
- or perform it immediately when allowed.

Result is reflected in Session/Diagnostics events, not in a direct
command response.

---

## 4. Events (Pulse → Aura)

### 4.1 Session State Events

#### **`session.statusUpdated`**

Current Session status snapshot.

Fields may include:

- `engineState` (`uninitialised`, `initialising`, `ready`, `running`,
  `suspended`, `safeMode`),
- `projectDirty` (bool),
- `backgroundTasksActive` (bool),
- `safeModeReason` (optional short string),
- `globalWarnings` (optional list of warning descriptors),
- optional placeholders for future integration:

  - `undoAvailable` / `redoAvailable` (bool),
  - `nextUndoLabel` / `nextRedoLabel` (short strings),
  - `scriptingStatus` (e.g. `disabled`, `enabled`, `error`).

Exact fields for undo/scripting can be refined when those domains are
formalised.

---

#### **`session.engineStateChanged`**

Event emitted whenever `engineState` changes, for quick UI response without
requiring a full status snapshot.

---

#### **`session.dirtyStateChanged`**

Emitted whenever the project dirty flag changes.

Fields:

- `projectDirty` (bool).

Aura uses this for save prompts and window title updates.

---

### 4.2 Background Task Events

#### **`session.backgroundTasksUpdated`**

Summarises background activity.

Fields may include:

- `active` (bool),
- list of tasks with:

  - `taskId`,
  - `kind` (e.g. `analysis`, `render`, `mediaRelink`, `composerSync`),
  - `status` (`pending`, `running`, `completed`, `failed`),
  - optional progress estimate.

Detailed task information belongs to the relevant domain; Session only
maintains a coarse overview.

---

### 4.3 Global Warning Events

#### **`session.warningAdded`**

A global warning has been raised.

Fields:

- `warningId`,
- `severity` (`info`, `warning`, `error`),
- `sourceDomain` (e.g. `hardware`, `media`, `plugin`, `engine`),
- `message` (short human-readable string).

---

#### **`session.warningCleared`**

A previously raised warning has been cleared.

Fields:

- `warningId`.

Aura uses these to update global warning indicators and to manage a warnings
panel.

---

## 5. Snapshot Semantics

Project snapshots include:

- the project dirty state at the time of snapshot creation?  
  (Typically snapshots are created *because* of state changes, so
  snapshotting the dirty flag is less critical.)

More importantly, Session-derived information that *comes from other domains*
(e.g. presence of missing plugins, missing media) is fully captured by the
state of those domains within the snapshot.

Session itself does not define its own separate snapshot data structure;
it is a projection over other domain states.

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

Undo/redo snapshot semantics will be defined in the Undo domain and will
integrate with Session's reporting fields (`undoAvailable`, etc.).

---

## 6. Relationship to Diagnostics, Undo and Scripting

- **Engine Diagnostics**  
  Diagnostics provide detailed performance and error information. Session
  provides a simplified, user-facing summary. For example:

  - Diagnostics may record “plugin X crashed”.
  - Session may show “Safe mode enabled due to plugin failure”.

- **Undo/Redo (future)**  
  Undo domain will define how actions are recorded and reverted. Session
  will surface the high-level status (availability and labels) so Aura can
  present undo/redo UI without querying the undo stack directly.

- **Scripting/Automation (future)**  
  A scripting domain may define script execution and automation logic.
  Session will reflect whether scripting is enabled, whether scripts are
  currently running, and whether there are script-level errors that affect
  the session.

Session’s role is to provide a **stable, compact summary** of the DAW’s
overall state for UI and integration, without duplicating the responsibilities
of specialised domains.
