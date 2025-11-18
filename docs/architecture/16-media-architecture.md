# Media Architecture

This document defines the architecture for media management, browsing and
discovery in Loophole.

It focuses on:

- how **media assets** (audio, MIDI, video, analysis) are represented in Pulse,
- how **storage backends** (local, cloud-backed, external drives) are handled,
- how **availability** and **missing/offline** states are modelled safely,
- how the **Media Library** and **Media Browser** are structured conceptually,
- how **kits** and **pad workspaces** are represented and populated,
- how **Composer** augments media with learned metadata and recommendations.

Implementation details for IPC commands/events are covered by the Pulse Media
domain specification; this document describes the higher-level architecture and
design goals.

---

## 1. Goals

Media handling in Loophole must:

1. Treat media as a **first-class creative resource**, not just files in
   directories.
2. Never block the audio engine or UI on slow or unreliable storage (cloud,
   network, external drives).
3. Provide a rich, expressive **Media Browser** that supports:
   - fast, goal-driven search,
   - playful discovery and experimentation,
   - multiple visual browsing paradigms.
4. Robustly handle **missing, offline or remote media** without corrupting
   projects or destroying metadata.
5. Provide a model for **kits** and **pad workspaces** that can be generated
   intelligently from metadata and Composer.
6. Cleanly separate:
   - **global library** (all user media),
   - **project library** (media used in a project),
   while allowing smooth navigation between them.

---

## 2. Core Concepts

### 2.1 Media Items

A **Media Item** is Pulse’s canonical representation of a media asset.

Key properties:

- `mediaId` – stable per-project identifier.
- `kind` – `audio`, `midi`, `video`, `analysis`, or future extensions.
- `name` – human-readable label (user-editable).
- `origin` – how the media arrived:
  - recorded,
  - imported,
  - generated (rendering/bounce),
  - external reference, etc.
- `storage` descriptor:
  - which storage backend,
  - project-relative or absolute path,
  - optional remote URL or provider-specific ID.
- `format` metadata (for audio):
  - container type,
  - sample rate,
  - channel count,
  - bit depth / float,
  - duration.
- `status` (see 2.2).
- `analysis` references (see 2.5).
- `tags` and `attributes`:
  - user tags,
  - Composer-assigned tags (instrument, mood, genre),
  - derived attributes (BPM, key, spectral features).

Media Items are **referenced** by Clips and other model objects via `mediaId`.
They are never addressed by raw file path in core project data.

---

### 2.2 Availability and Status

Every Media Item has a **status** describing both logical lifecycle and
physical availability. Example states:

- Lifecycle:
  - `pending` – reserved for recording/rendering, not finalised.
  - `final` – fully registered, referenced by Clips or Kits.
  - `abandoned` – allocated but never used (eligible for cleanup).
- Availability:
  - `local` – file is present and readable on local storage.
  - `remote-fetching` – file exists on a cloud-backed filesystem but is being
    fetched (e.g. Dropbox “online-only” file being hydrated).
  - `remote-offline` – file is remote-only and not currently retrievable.
  - `external-online` – file is on a currently mounted external drive.
  - `external-offline` – file is on an external drive that is not mounted.
  - `missing` – file not found at expected location and no known alternative.

Pulse combines lifecycle + availability into a coherent status model exposed to
Aura via the Media domain and Session warnings.

**Key rule:** media availability must never be assumed. All operations involving
file I/O are asynchronous and cancellable.

---

### 2.3 Storage Backends

Loophole supports multiple **storage backends** for media:

- **Project-local storage**:
  - dedicated `Media/` tree within the project directory (e.g. `Audio/`,
    `Video/`, `Analysis/`).
  - preferred for long-term stability and portability.

- **Cloud-backed local filesystems**:
  - e.g. Dropbox, OneDrive, iCloud Drive.
  - appear as normal paths but may page files out or hydrate them lazily.
  - Loophole must treat them as potentially **slow, asynchronous** mediums.

- **External volumes**:
  - USB drives, network mounts, etc.
  - may appear and disappear between sessions.
  - paths may change (e.g. different drive letters or mount points).

- **Purely external references**:
  - files that live outside the project tree and are intentionally not copied
    in (e.g. shared sample libraries).

Each Media Item’s `storage` descriptor identifies:

- backend type,
- current resolved path (if any),
- any indirect IDs (e.g. provider-specific handles).

---

### 2.4 Project Library vs Global Library

There are two main scopes for media:

- **Project Library**:
  - the subset of Media Items actually referenced by the project:
    - Clips,
    - Track/Lane references,
    - Freeze and Bounce results,
    - kits committed to the project.
  - fully under Loophole’s snapshot and undo/redo model.
  - expected to be portable and stable over time.

- **Global Library**:
  - the user’s broader pool of media resources:
    - scanned/sample folders,
    - packs and products,
    - user sample libraries.
  - managed by a **Library Indexer** in Pulse (or a companion service),
    storing metadata but not necessarily copying files into projects.

Aura’s Media Browser can seamlessly switch between:

- “Project media only” views,
- “Global library” views,
- combined/project-focused views (“show similar sounds from global library”).

Integration with Composer allows global library indexing and tagging to be
shared and improved across users.

---

### 2.5 Analysis Artefacts

**Analysis Artefacts** are derived data products associated with Media Items, such as:

- waveform overviews,
- loudness envelopes,
- beat grids / transient markers,
- spectral summaries,
- embeddings for similarity search.

Each artefact:

- may be stored as:
  - a sidecar media object (`analysis` kind),
  - or an inline blob in an analysis cache.
- references:
  - the owning `mediaId`,
  - the `analysisType` and parameters used.

Signal is responsible for heavy audio analysis; Pulse indexes and orchestrates.

Aura uses analysis artefacts to:

- render thumbnails and waveforms,
- drive smart browsing and similarity views,
- assist with beat-matching and time-stretch preview.

---

## 3. Remote and Offline Storage Handling

### 3.1 Cloud-Backed Filesystems (e.g. Dropbox)

Cloud-backed systems (Dropbox, iCloud, etc.) may:

- report files as present in directory listings,
- but only hydrate file contents on read,
- potentially block reads for many seconds.

Loophole must treat these as **untrusted for latency**:

- All media reads go through an asynchronous I/O layer.
- Any blocking call is done on background threads, never on the audio thread
  and preferably not on the main UI thread either.
- Media status transitions:
  - `remote-offline → remote-fetching → local`,
  - with progress and errors reported via Media and Session events.

If a provider is hydrating files too slowly:

- Loophole must allow users to cancel individual operations.
- Background tasks are surfaced in the Session domain (background tasks panel).
- The project remains fully usable while long-running downloads occur.

---

### 3.2 External Drives

Media on removable or external volumes is treated similarly:

- When a drive is mounted:
  - Media Items whose storage targets that volume may transition to
    `external-online` or `local`.
- When a drive is unmounted:
  - affected Media Items become `external-offline`,
  - Clips referencing them fall back to **placeholders**.

Critically:

- Saving a project while a drive is offline must **not** destroy references to
  those Media Items.
- The project file must retain:
  - `mediaId`,
  - full metadata,
  - original storage descriptor,
  - any tags and analysis information already known.

When the drive returns (or the user relinks to a new path), media becomes
available again without breaking the project or losing context.

---

### 3.3 Missing Media and Placeholders

If a Media Item cannot be found:

- status is set to `missing`,
- but:
  - `mediaId`,
  - tags,
  - usage references,
  - analysis artefacts (if any)
  are kept intact.

Clips referencing missing media:

- render as **placeholders** in Aura (visually distinct),
- can be relinked to new media,
- or used as context to fetch **suggested replacements** (“similar sounds”)
  via Composer and global library search.

The architecture assumes that **missing media is recoverable**; it is never
“forgotten” unless the user explicitly removes or replaces it.

---

## 4. Media Browser and Library Model

### 4.1 Browser Modes

Aura’s Media Browser can present several **browsing modes**, all backed by the
same Media model in Pulse:

- **List / Table view**:
  - classical list with columns for name, BPM, key, length, tags, etc.
- **Waveform thumbnail view**:
  - grid or strip view with mini waveforms / spectrograms.
- **Visual Cluster (“point cloud”) view**:
  - XO-like 2D (or 3D) embedding space:
    - each point is a Media Item,
    - clustered by similarity,
    - coloured by type or timbre.
- **Hypergrid / Drum Grid view**:
  - rows as semantic families (kick/snares/hats/textures),
  - columns as qualities (deep/bright/punchy/etc.),
  - dynamic based on tags and embeddings.
- **Folder/pack view**:
  - virtual representation of folder/pack structure for users who prefer it.
- **Project view**:
  - only media used in current project, grouped by Track, Section, or usage.

These modes are purely Aura concerns, but the architecture ensures that:

- metadata needed for each view is available via the Media + Composer domains,
- views can share selection, filters and context state.

---

### 4.2 Filters, Facets and Search

Search must be able to filter Media Items by:

- textual fields:
  - name, pack name, origin, user notes;
- attributes:
  - BPM, BPM range,
  - key, relative key compatibility,
  - duration;
- categorical tags:
  - drums, bass, pad, vocal, FX, etc.;
- timbral/mood tags:
  - warm, harsh, airy, noisy, dark, bright, etc.;
- usage:
  - used in this project,
  - used in selected Track or Section,
  - unused in this project;
- availability:
  - local only,
  - remote,
  - offline,
  - missing.

Composer augments search with **semantic similarity**:

- “sounds like this one”,
- “often used after sounds like this in similar projects”,
- “common choices for intros/choruses/vocal layers”.

Search and filtering logic is implemented in Pulse, with Composer consulted for
semantic ranking and suggestions.

---

### 4.3 Preview and Audition

Previewing is a joint responsibility of Aura, Pulse and Signal:

- Aura initiates preview requests:
  - simple preview (solo playback),
  - audition-in-context (play with current project loop),
  - preview-through-chain (route through selected Channel/Node chain).
- Pulse:
  - decides routing (direct vs through existing Channels),
  - ensures preview does not mutate project state.
- Signal:
  - performs audio decoding and playback,
  - handles time-stretching to project tempo,
  - handles pitch-shifting to project key if requested,
  - ensures preview is quantised to grid when required.

Preview playback must be:

- non-invasive (not altering transport unless desired),
- low-latency, but allowed to fail gracefully if media is remote/offline,
- cancellable instantly without artefacts.

---

## 5. Kits and Pad Workspaces

### 5.1 Kit Model

A **Kit** is a structured collection of assignable slots mapped to media or
instrument presets.

Examples:

- **Drum kits**:
  - slots: `kick`, `snare`, `clap`, `hat`, `perc1..N`, `fx`.
- **Vocal kits**:
  - slots: `lead`, `adlib`, `chop1..N`, `fx`.
- **Texture kits**:
  - slots: `drone`, `swells`, `risers`, `impacts`.

Each slot references either:

- a Media Item (`mediaId`), or
- an instrument/preset definition (for instrument Nodes).

Kit properties:

- `kitId`,
- `name`,
- `scope`:
  - `global` (usable in any project),
  - `project` (saved with this project),
  - `ephemeral` (workspace-only, not yet committed),
- `tags`:
  - genre/style, mood, usage (intro/verse/drop, etc.),
- `mapping` to pad coordinates if used in a Pad Workspace.

Kits are stored in Pulse and may be shared or suggested via Composer (e.g.
“kits that people often build around sounds like this”).

---

### 5.2 Pad Workspaces

A **Pad Workspace** is an interactive environment for experimentation with kits
outside the strict Track/Clip view.

Key features:

- surfaces like a hardware controller (e.g. 4×4, 8×8 pad grid),
- each pad mapped to:
  - a Kit slot, or
  - a direct Media Item,
- supports:
  - real-time playing via MIDI or mouse,
  - on-the-fly swapping of sounds,
  - intelligent randomisation.

Pad Workspaces are:

- **separate from the main timeline**, but
- can **commit** patterns or clips into Tracks/Lanes:
  - e.g. “commit last 4 bars as MIDI to Track X”,
  - or “render this pad performance as a Clip in Track Y”.

Pad Workspaces may have their own local state stored in Pulse, for example:

- active Kit,
- quantisation settings,
- pad layout,
- current pattern memory.

---

### 5.3 Intelligent Kit Building

The **Kit Builder** is a conceptual component that uses:

- project context:
  - BPM, key, timebase,
  - active Tracks and their roles,
  - Sections (intro, verse, chorus, drop),
- user constraints:
  - desired tags (e.g. “lofi”, “cinematic”, “minimal”),
  - preferred instrument categories,
  - exclusions,
- Composer data:
  - embeddings and similarity,
  - common combinations,
  - popularity of certain samples in similar contexts,
- local history:
  - user’s favoured choices,
  - prior kits and patterns.

to automatically populate a Kit.

Examples:

- “Generate a drum kit for this project”:
  - picks kick/snare/hat/percs in compatible style, tuned as needed.
- “Build a vocal chop kit from this acapella”:
  - detects candidate regions, slices them, assigns to pads.
- “Create a kit for this Section (Chorus)”:
  - chooses loops and stabs that work harmonically and rhythmically there.

The Kit Builder runs in Pulse (possibly with Composer assistance), and exposes
its operations via:

- IPC (for Aura to request kits),
- media and kit domain changes (new kits, new media items where needed).

---

## 6. Integration with Composer

Composer is a separate service that provides **shared knowledge** and
recommendations. Media integration includes:

- **Metadata learning**:
  - instrument classification,
  - BPM/key inference,
  - mood / timbre tags,
  - pack/source recognition.
- **Embedding space**:
  - multi-dimensional vectors for each Media Item,
  - used for similarity search and visual cluster views.
- **Collective behaviour**:
  - which sounds tend to be used together,
  - common stackings and layering patterns,
  - kit layouts that work well in certain genres.

Pulse synchronises Media metadata with Composer (subject to privacy settings),
and caches Composer’s responses locally for offline use.

Aura uses Composer-enhanced metadata for:

- smart search queries,
- recommendations,
- “more like this” actions,
- visual embedding-based navigation.

---

## 7. Media Lifecycle and Project Safety

### 7.1 Reserve / Finalise Pattern

Recording and rendering use a common **reserve/finalise** pattern:

1. Pulse asks Media to **reserve** a target:
   - creates a `pending` Media Item with an allocated path.
2. Signal records or renders into that path.
3. Upon success, Pulse **finalises** the item:
   - marks it `final`,
   - populates duration/format,
   - attaches it to Clips, Kits, or Freeze results.
4. On cancellation or failure, Pulse may:
   - mark as `abandoned` (eligible for cleanup), or
   - delete the file based on configuration.

This pattern ensures no heavy I/O happens synchronously on the main thread, and
no partially written media is treated as stable.

---

### 7.2 Save, Load and Draft Behaviour

On `project.save` / `project.saveAs`:

- only `final` Media Items are considered part of the stable project state.
- references to missing or offline media are preserved as-is; they are *not*
  automatically removed or overwritten.

On `project.saveDraft`:

- Pulse may persist additional transient state (e.g. pending media, Pad
  Workspace contents) to aid crash recovery or continuity,
- but these drafts are not required to be portable between machines.

On project load:

- Pulse rebuilds the Media Pool from the project file,
- reconciles availability via storage backends,
- marks items as `local`, `external-online`, `external-offline`, `remote-*`,
  or `missing` as appropriate,
- does *not* attempt to automatically alter user paths beyond basic
  platform-specific path adjustments.

---

## 8. Relationship to Other Architecture Documents

- **Tracks, Channels and Lanes (06)**  
  Clips in Lanes reference Media Items via `mediaId`. The Media architecture
  defines how those Media Items are stored and located.

- **Clips (07)**  
  Clips represent timeline placement and editing of Media. Media defines the
  underlying content; Clips define how it is used.

- **Parameters and Automation (08)**  
  Media parameters (e.g. gain, pitch, start/end) may be exposed to Automation
  but do not change core Media identity.

- **Nodes (09)**  
  Nodes may use Media Items as sources (e.g. samplers, granular processors).
  The Media architecture ensures Node configurations reference Media in a
  stable way.

- **Processing Cohorts and Anticipative Rendering (10)**  
  Rendering and analysis tasks depend on Media I/O. Cohort logic ensures that
  no heavy media operation disrupts real-time processing.

- **Session and History**  
  Background tasks for media (import, analysis, remote fetches) are surfaced
  via Session. Media-related operations are undoable where appropriate via
  the History domain.

- **Composer (05)**  
  Composer sits alongside Media as the shared knowledge service, providing
  inferred metadata and recommendations. This document identifies where that
  integration lives conceptually.

The Media architecture provides the backbone for how Loophole understands,
stores, and creatively navigates media, enabling a Media Browser that is both
technically robust and deeply inspiring to use.
