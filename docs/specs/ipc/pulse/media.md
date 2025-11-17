# Pulse Media Domain Specification

This document defines the Media domain of the Pulse project protocol.  
It covers commands and events for managing **media assets** used by the
project, including:

- audio files (recorded and imported),
- MIDI clip data (as persistent media objects),
- analysis artefacts (waveform overviews, loudness, etc.),
- media pool indexing,
- missing-media detection and relinking.

Pulse maintains a logical **Media Pool** for each project.  
Signal performs low-level I/O and analysis.  
Aura presents and manages media via the Media Pool model.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Media Concepts](#2-media-concepts)  
  - [2.1 Media Items and IDs](#21-media-items-and-ids)  
  - [2.2 Media Kinds](#22-media-kinds)  
  - [2.3 Media Storage Layout](#23-media-storage-layout)  
  - [2.4 Analysis Artefacts](#24-analysis-artefacts)  
  - [2.5 Media Pool and References](#25-media-pool-and-references)  
  - [2.6 Media Status](#26-media-status)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Media Pool Queries](#31-media-pool-queries)  
  - [3.2 Import and Registration](#32-import-and-registration)  
  - [3.3 Recording Integration](#33-recording-integration)  
  - [3.4 Relinking and Relocation](#34-relinking-and-relocation)  
  - [3.5 Analysis and Waveform Generation](#35-analysis-and-waveform-generation)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Media Pool Events](#41-media-pool-events)  
  - [4.2 Relink and Relocation Events](#42-relink-and-relocation-events)  
  - [4.3 Analysis Events](#43-analysis-events)  
- [5. Snapshot and Persistence Semantics](#5-snapshot-and-persistence-semantics)  
- [6. Realtime and Safety Considerations](#6-realtime-and-safety-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

The Media domain provides a coherent model for all media assets associated
with a project:

- manages the **Media Pool** (set of known media items),
- assigns stable media IDs,
- tracks file paths and relocation,
- orchestrates analysis tasks (waveforms, loudness, etc.),
- integrates with Recording and Bounce/Freeze operations.

Goals:

- Media references in Clips/Tracks are **stable** and survive file moves,
- missing media is detected and clearly reported,
- importing and recording audio feeds automatically into the Media Pool,
- analysis happens in the background with clear status reporting.

---

## 2. Media Concepts

### 2.1 Media Items and IDs

Each media asset is represented as a **Media Item** with a stable ID:

- `mediaId` – unique within a project,
- metadata such as:

  - kind (`audio`, `midi`, `video`, `analysis`, etc.),
  - human-readable name,
  - file path (project-relative where possible),
  - duration (musical and/or absolute),
  - sample rate, channel count (for audio),
  - format information.

Clips and other domains reference media via `mediaId`, not by file path.

---

### 2.2 Media Kinds

Typical media kinds include:

- `audio` – audio files (WAV, FLAC, etc.),
- `midi` – persistent MIDI sequences (for clip content),
- `video` – video files, if/when supported,
- `analysis` – files or blobs containing precomputed analysis (waveforms,
  spectral data, loudness curves, etc.),
- `other` – placeholders for future extensions.

MIDI clip contents may be embedded directly in the project model for
simplicity, but the Media domain remains capable of treating them as media if
we later externalise them.

---

### 2.3 Media Storage Layout

Media is typically stored in a project-specific structure, e.g.:

- `Audio/` – recorded and imported audio files,
- `Video/` – imported video,
- `Analysis/` – waveform overviews and similar,
- `External/` – references to external media not copied into the project.

The precise layout is not specified here, but the Media domain:

- keeps track of where each item lives,
- prefers **project-relative paths**,
- can support both “copy into project” and “reference in place” modes for
  imported files (subject to user preferences).

---

### 2.4 Analysis Artefacts

Analysis artefacts are derived from primary media, e.g.:

- peak files / waveform overviews,
- loudness envelopes,
- spectral summaries.

Each analysis artefact:

- links to a primary `mediaId`,
- has a specific analysis type,
- may be stored as a sidecar file or in an optimised cache.

Signal performs the heavy lifting; Media coordinates and indexes results.

---

### 2.5 Media Pool and References

The **Media Pool** is the set of all Media Items known to the project.

- Clips and other objects reference media via `mediaId`,
- the Media Pool tracks which items are:

  - present and readable,
  - missing (file not found),
  - relinked (file moved but found again),
  - offline (e.g. network drives).

Pulse owns the Media Pool and ensures consistency across changes.

---

### 2.6 Media Status

Each Media Item has a lifecycle status, for example:

- `pending` – reserved as a recording/render target but not yet finalised.
- `final` – complete and referenced by Clips, renders or other domains.
- `abandoned` – optional internal state representing media that was reserved
  but never used.

Only `final` media is considered part of the stable project model. Pending and
abandoned items are treated as transient and may be cleaned up by maintenance
routines.

---

## 3. Commands (Aura → Pulse)

### 3.1 Media Pool Queries

#### **`media.listItems`**

Request a summary of Media Items in the pool.

Fields (all optional filters):

- `kind` (e.g. `audio`, `midi`),
- `status` (e.g. `ok`, `missing`, `offline`).

Pulse responds via `media.itemsUpdated` (or a dedicated response envelope).

---

#### **`media.getItemDetails`**

Request detailed information for a specific `mediaId`.

Fields:

- `mediaId`.

Pulse responds with extended metadata (format, duration, path, etc.).

---

### 3.2 Import and Registration

#### **`media.importFile`**

Import an external file into the project as a Media Item.

Fields:

- `sourcePath` (filesystem path from Aura’s perspective),
- `kindHint` (optional, e.g. `audio`, `video`),
- `copyMode`:

  - `"copyIntoProject"` – copy the file into the project’s media folder,
  - `"referenceExternal"` – reference the media in its existing location.

Pulse/Signal:

- validate the file,
- copy/convert as needed,
- create a new Media Item with `mediaId`,
- emit `media.itemAdded`.

---

#### **`media.registerExistingFile`**

Register a file that already resides in the project’s media directory.

Fields:

- `relativePath` (project-relative),
- optional metadata overrides (name, tags).

Used when projects are migrated or when media already exists in the media
folder.

---

### 3.3 Recording Integration

Recording domain focuses on *what* to record; Media must manage the resulting
files.

#### **`media.reserveRecordingTarget`**

Request that Signal prepare a file for recording.

Fields may include:

- `kind` (`audio` for now),
- `nameHint`,
- optional folder/category hint.

Pulse/Signal:

- allocate a file path within the project’s media structure,
- create a provisional Media Item (status = `pending`),
- return `mediaId` and path to Signal for recording.

---

#### **`media.finaliseRecording`**

Mark a reserved Media Item as complete.

Fields:

- `mediaId`,
- optional final metadata (duration, sample rate, channels) if not already
  populated.

Pulse updates Media Pool and emits `media.itemUpdated`.  
Recording domain then attaches this media to Clips as needed.

---

#### **`media.discardRecording`**

Discard a reserved but incomplete or cancelled recording.

Fields:

- `mediaId`,
- optional flag indicating whether to delete the file on disk.

Useful for aborted takes.

---

### 3.4 Relinking and Relocation

#### **`media.relocateItem`**

Update the file path for a Media Item.

Fields:

- `mediaId`,
- new path fields:

  - `relativePath` (preferred),
  - or absolute `path`.

Pulse:

- validates the file’s existence,
- updates Media Item status,
- emits `media.itemUpdated`.

---

#### **`media.relinkMissing`**

Attempt to relink missing media items.

Fields may include:

- `searchRootPaths` (one or more directories to scan),
- optional `mediaIds` subset.

Signal performs a search; Pulse updates the Media Pool with new locations and
emits `media.itemsUpdated` plus detailed relink events.

---

### 3.5 Analysis and Waveform Generation

#### **`media.requestAnalysis`**

Request analysis for a Media Item.

Fields:

- `mediaId`,
- `analysisType` (e.g. `waveform`, `loudness`, `spectrumOverview`),
- optional parameters (resolution, channel behaviour, etc.).

Signal schedules background tasks to generate analysis artefacts.

---

#### **`media.cancelAnalysis`**

Cancel an outstanding analysis request.

Fields:

- `mediaId`,
- optional `analysisType` (if multiple analyses are possible per item).

---

## 4. Events (Pulse → Aura)

### 4.1 Media Pool Events

#### **`media.itemsUpdated`**

Media Pool updated (e.g. after import, recording, relink, batch edit).

Fields:

- summary list of items or a minimal “version” counter,
- Aura may re-query specifics via `media.listItems` / `media.getItemDetails`.

---

#### **`media.itemAdded`**

A new Media Item has been created.

Fields:

- `mediaId`,
- minimal metadata (name, kind, status).

---

#### **`media.itemUpdated`**

Metadata or status for a Media Item has changed.

Fields:

- `mediaId`,
- key updated properties (status, path, analysis availability, etc.).

---

#### **`media.itemRemoved`**

A Media Item has been removed from the pool.

Fields:

- `mediaId`.

Deletion semantics (and safety) are controlled by settings; Media IPC only
reports the result.

---

### 4.2 Relink and Relocation Events

#### **`media.relinkStarted`**

A relink/search operation has begun.

---

#### **`media.relinkProgress`**

Periodic progress updates for a relink operation.

Fields:

- number of items scanned,
- number of items found,
- search locations visited.

---

#### **`media.relinkCompleted`**

Summary of a relink operation.

Fields:

- number of items successfully relinked,
- number of items still missing.

---

### 4.3 Analysis Events

#### **`media.analysisStarted`**

Analysis started for a `mediaId` and `analysisType`.

---

#### **`media.analysisProgress`**

Progress update for an analysis operation.

Fields:

- `mediaId`,
- `analysisType`,
- optional progress estimate (0–1).

---

#### **`media.analysisCompleted`**

Analysis has finished successfully.

Fields:

- `mediaId`,
- `analysisType`,
- reference to the analysis artefact (analysis mediaId or inline descriptor).

---

#### **`media.analysisFailed`**

Analysis failed.

Fields:

- `mediaId`,
- `analysisType`,
- error description.

Aura uses these events to:

- show waveform/thumbnails once ready,
- update loudness meters in offline analyses,
- display error badges when analysis fails.

---

## 5. Snapshot and Persistence Semantics

Project snapshots include the Media Pool **metadata**:

- list of Media Items (`mediaId`, kind, name, status),
- project-relative paths for files,
- analysis availability flags.

Snapshots do **not** include:

- the actual media files (audio/MIDI/video),
- the contents of analysis artefacts.

Those remain on disk. Snapshot application uses `mediaId` and paths to refer
to media.

Missing-media state is a function of:

- Media Pool metadata,
- actual files present on disk when the project is opened.

When applying snapshots, Pulse must:

- not delete media items that are still referenced in other snapshots,
- maintain consistent `mediaId` references.

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

### Pending Media and Project Save/Load

Pending Media Items (`status = pending`) are **not** included in committed
project snapshots intended for long-term storage:

- `project.save` and `project.saveAs` serialise only `final` media references.
- Pending items exist only in the live in-memory model and in the draft state.

On project load:

- Pulse does not recreate pending Media Items from the project file.
- Any stray files on disk created for abandoned recordings or renders may be
  detected by a maintenance pass and either:

  - offered to the user for recovery (future enhancement), or
  - cleaned up automatically depending on host settings.

Draft save behaviour (`project.saveDraft`) may persist pending Media Items for
crash recovery, but these are treated as ephemeral and are not required to be
restorable across sessions.

---

## 6. Realtime and Safety Considerations

Media operations must not interfere with realtime audio:

- file copying, importing, and analysis are done on background threads,
- heavy operations are rate-limited and can be paused or cancelled.

Recording uses a **reserve/finalise** pattern to avoid blocking:

- files are prepared and pre-allocated where possible,
- actual recording writes to pre-reserved media,
- finalisation happens after the record pass.

Media searches (for relinking) and directory scans must also be executed in
the background and surfaced via progress events.

---

## 7. Relationship to Other Domains

The Media domain interacts with:

- **Recording**  
  Recording uses `media.reserveRecordingTarget` and `media.finaliseRecording`
  to produce new Media Items.

- **Bounce / Freeze / Render**  
  Offline renders produce new Media Items (audio files). Media coordinates
  their registration and analysis.

- **Clips and Lanes**  
  Clips reference `mediaId` to associate timeline regions with media.
  Media Pool must be consistent with Clip usage.

- **Engine Diagnostics and Session**  
  Long-running media and analysis tasks appear in Session background tasks
  and may be referenced in Engine Diagnostics messages when relevant.

- **Composer**  
  Composer may collect anonymised statistics about media usage (e.g. common
  sample rates, typical recording practices) for future recommendations,
  though this is optional and outside the core Media IPC.

The Media domain provides the backbone for how Loophole manages media
assets, enabling robust recording, import, bounce and analysis workflows.
