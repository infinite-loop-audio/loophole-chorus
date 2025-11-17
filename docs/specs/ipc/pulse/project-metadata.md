# Pulse Project Metadata Domain Specification

This document defines the Project Metadata domain of the Pulse project
protocol.

It covers high-level, non-structural information about a project, including:

- project identity and descriptive metadata,
- key signature and musical context,
- global markers and sections on the timeline,
- project-level tags and colour themes.

Unlike the Project domain (which controls loading/saving and structural
content), the Metadata domain focuses on *describing* the project and its
timeline.

Pulse owns the metadata model; Aura presents and edits it.  
Signal may consult some metadata (e.g. key signature) but is not required to.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Metadata Concepts](#2-metadata-concepts)  
  - [2.1 Project Identity](#21-project-identity)  
  - [2.2 Musical Context and Key Signature](#22-musical-context-and-key-signature)  
  - [2.3 Global Markers](#23-global-markers)  
  - [2.4 Sections and Regions](#24-sections-and-regions)  
  - [2.5 Tags, Colours and Classification](#25-tags-colours-and-classification)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Project Identity](#31-project-identity)  
  - [3.2 Key Signature and Musical Context](#32-key-signature-and-musical-context)  
  - [3.3 Markers](#33-markers)  
  - [3.4 Sections](#34-sections)  
  - [3.5 Tags and Colours](#35-tags-and-colours)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Project Identity Events](#41-project-identity-events)  
  - [4.2 Musical Context Events](#42-musical-context-events)  
  - [4.3 Marker Events](#43-marker-events)  
  - [4.4 Section Events](#44-section-events)  
  - [4.5 Tag/Colour Events](#45-tagcolour-events)  
- [5. Snapshot and Persistence Semantics](#5-snapshot-and-persistence-semantics)  
- [6. Relationship to Other Domains](#6-relationship-to-other-domains)

---

## 1. Overview

This domain provides a structured way to represent:

- who and what the project is for,
- where important points and sections lie on the timeline,
- the musical key and related context,
- global tags and colour schemes for organisation.

Key goals:

- keep metadata orthogonal to structural edits (Tracks, Clips, etc.),
- make markers and sections first-class, not hacked into tempo maps,
- provide a clean surface for future features (e.g. chord lanes, advanced
  browsing, Composer integration).

---

## 2. Metadata Concepts

### 2.1 Project Identity

Project identity covers basic descriptive fields:

- `title` – human-readable project title,
- `subtitle` or `version` (optional),
- `author` and optional `contributors`,
- `description` / notes (free-form text),
- `creationDate`, `lastModifiedDate` (informational, not authoritative),
- optional external identifiers (e.g. catalogue numbers, ISRC/ISWC for
  full export workflows later).

These fields are not required for engine operation but are important for:

- library browsing,
- export metadata,
- user orientation.

---

### 2.2 Musical Context and Key Signature

Musical context includes:

- **Global key signature**:

  - tonic (e.g. C, Eb),
  - mode (e.g. major, minor, dorian),
  - optional scale name.

- Optional **key changes** over time (similar to tempo/signature markers),
  though many projects will use a single global key.

Key signatures are used by:

- piano roll (highlighting scale tones),
- chord/groove lanes (later),
- media library queries (e.g. “find loops in compatible keys”).

This spec treats key signature as metadata with an optional timeline overlay;
harmonic rules and chord lanes live elsewhere.

---

### 2.3 Global Markers

Global markers are timeline annotations independent of tempo markers:

- position in **musical time** (bar/beat) as primary,
- optional absolute time cache for convenience.

Marker types might include:

- `generic` – vanilla marker, just name and position,
- `locator` – general location markers (A/B style),
- `cue` – film/game cues or hitpoints,
- `note` – arbitrary notes at specific times.

Each marker has:

- `markerId`,
- `position` (musical),
- `name`,
- optional `type`,
- optional `colourId`,
- optional free-form `comment`.

Markers are global; they are not tied to any particular Track or Lane.

---

### 2.4 Sections and Regions

Sections represent larger structural regions on the timeline:

- `sectionId`,
- `startPosition`, `endPosition` (musical),
- `name` (e.g. “Verse 1”, “Chorus”, “Drop”),
- optional `type` (`intro`, `verse`, `chorus`, `bridge`, etc.),
- optional `colourId`,
- optional tags.

Sections are distinct from Clips:

- Clips represent actual media/material,
- Sections represent conceptual arrangement structure.

Potential future use:

- arrangement view / pattern-based workflows,
- “jump to section” transport actions,
- Composer guidance (common patterns, section templates).

---

### 2.5 Tags, Colours and Classification

Project-wide tags and colours support:

- classification (e.g. “mix pending”, “client approved”),
- visual organisation (colour coding of Tracks, Sections, Markers, etc.),
- library queries (“show projects tagged ‘lo-fi’”).

The Metadata domain manages:

- a palette of **colour definitions** (IDs mapped to RGB/HSLA),
- a list of **project tags** (strings or structured labels),
- optional per-object tag attachments (Markers/Sections referencing tags by ID).

Other domains (Tracks, Clips, etc.) may also reference these colour/tag IDs
for consistency.

---

## 3. Commands (Aura → Pulse)

### 3.1 Project Identity

#### **`metadata.setProjectInfo`**

Set one or more project identity fields.

Fields (all optional, partial updates allowed):

- `title`,
- `subtitle`,
- `author`,
- `contributors` (list of strings),
- `description`,
- optional external identifiers (simple key-value map).

Pulse:

- updates stored metadata,
- emits `metadata.projectInfoChanged`.

---

#### **`metadata.getProjectInfo`**

Request current project metadata.

Pulse responds via a dedicated response envelope or `metadata.projectInfoChanged`.

---

### 3.2 Key Signature and Musical Context

#### **`metadata.setGlobalKey`**

Set or clear the global key signature.

Fields:

- `key` (e.g. `C`, `Eb`, `F#`),
- `mode` (e.g. `major`, `minor`, `dorian`, `mixolydian`),
- optional `scaleName` (e.g. `pentatonic`, `harmonicMinor`),
- or `clear: true` to remove explicit key.

---

#### **`metadata.addKeyChange`**

Optionally define a key change at a specific position.

Fields:

- `position` (musical),
- `key`, `mode`, optional `scaleName`.

Pulse creates a key marker with `keyChangeId` and emits
`metadata.keyChangeAdded`.

---

#### **`metadata.updateKeyChange`**

Update a key change marker.

Fields:

- `keyChangeId`,
- new `position`, `key`, `mode`, etc.

---

#### **`metadata.removeKeyChange`**

Remove a key change marker.

Fields:

- `keyChangeId`.

Key-change lanes are likely a future/advanced feature; basic support is
included here for compatibility.

---

### 3.3 Markers

#### **`metadata.addMarker`**

Create a marker.

Fields:

- `position` (musical),
- `name`,
- optional `type` (`generic`, `locator`, `cue`, `note`, etc.),
- optional `colourId`,
- optional `comment`.

Pulse:

- creates `markerId`,
- inserts into marker map,
- emits `metadata.markerAdded`.

---

#### **`metadata.updateMarker`**

Update marker properties.

Fields:

- `markerId`,
- any subset of:

  - `position`,
  - `name`,
  - `type`,
  - `colourId`,
  - `comment`.

Pulse updates and emits `metadata.markerUpdated`.

---

#### **`metadata.removeMarker`**

Remove a marker.

Fields:

- `markerId`.

Pulse emits `metadata.markerRemoved`.

---

#### **`metadata.clearMarkersInRange`**

Remove markers within a time range.

Fields:

- `startPosition`,
- `endPosition`,
- optional `types` filter.

Useful for general editing operations.

---

### 3.4 Sections

#### **`metadata.addSection`**

Create a section region.

Fields:

- `startPosition`,
- `endPosition`,
- `name`,
- optional `type` (`intro`, `verse`, `chorus`, etc.),
- optional `colourId`,
- optional tags.

Pulse creates `sectionId` and emits `metadata.sectionAdded`.

---

#### **`metadata.updateSection`**

Update a section.

Fields:

- `sectionId`,
- any subset of:

  - `startPosition`,
  - `endPosition`,
  - `name`,
  - `type`,
  - `colourId`,
  - tags.

Pulse emits `metadata.sectionUpdated`.

---

#### **`metadata.removeSection`**

Remove a section.

Fields:

- `sectionId`.

Pulse emits `metadata.sectionRemoved`.

---

#### **`metadata.reorderSections`**

Change the display/ordering of sections if there is any concept of logical
order separate from timeline order (optional; often not needed).

Fields:

- ordered list of `sectionId`s.

If unused, this command can be omitted in implementation.

---

### 3.5 Tags and Colours

#### **`metadata.defineColour`**

Define or update a colour in the project palette.

Fields:

- `colourId`,
- `colour` (colour representation: e.g. RGBA or HSLA fields),
- optional `name`.

Pulse emits `metadata.colourDefined`.

---

#### **`metadata.removeColour`**

Remove a colour definition.

Fields:

- `colourId`.

Objects referencing this `colourId` may fall back to defaults.

---

#### **`metadata.addTag`**

Add a project-level tag.

Fields:

- `tagId`,
- `label`,
- optional `description`.

Tags may be referenced by Markers/Sections/Tracks/Clips via `tagId`, not
stored redundantly.

---

#### **`metadata.updateTag`**

Update tag metadata.

Fields:

- `tagId`,
- `label` / `description`.

---

#### **`metadata.removeTag`**

Remove a tag from the project.

Fields:

- `tagId`.

Objects using this tag may either drop it or retain a textual copy, subject
to design decisions in their own domains.

---

## 4. Events (Pulse → Aura)

### 4.1 Project Identity Events

#### **`metadata.projectInfoChanged`**

Project identity changed.

Fields:

- updated identity object (`title`, `author`, etc.).

Aura updates project headers, window titles, project info panels.

---

### 4.2 Musical Context Events

#### **`metadata.globalKeyChanged`**

Global key signature changed or cleared.

Fields:

- key/mode/scaleName or null.

---

#### **`metadata.keyChangeAdded`**

Key change marker added.

---

#### **`metadata.keyChangeUpdated`**

Key change marker updated.

---

#### **`metadata.keyChangeRemoved`**

Key change marker removed.

---

### 4.3 Marker Events

#### **`metadata.markerAdded`**

Marker created.

---

#### **`metadata.markerUpdated`**

Marker edited.

---

#### **`metadata.markerRemoved`**

Marker deleted.

---

#### **`metadata.markersChanged`**

Optional coarse-grained event used when many markers are edited at once
(e.g. bulk delete or range shift). Aura can re-query marker maps.

---

### 4.4 Section Events

#### **`metadata.sectionAdded`**

Section created.

---

#### **`metadata.sectionUpdated`**

Section updated.

---

#### **`metadata.sectionRemoved`**

Section deleted.

---

#### **`metadata.sectionsChanged`**

Coarse-grained “many sections changed” event, similar to `markersChanged`.

---

### 4.5 Tag/Colour Events

#### **`metadata.colourDefined`**

Colour palette entry created/updated.

---

#### **`metadata.colourRemoved`**

Colour palette entry removed.

---

#### **`metadata.tagAdded`**

Tag added.

---

#### **`metadata.tagUpdated`**

Tag updated.

---

#### **`metadata.tagRemoved`**

Tag removed.

---

## 5. Snapshot and Persistence Semantics

Project snapshots include:

- project identity fields,
- global key signature and key changes,
- marker map (all markers),
- section map (all sections),
- tag and colour definitions.

Applying a snapshot:

- replaces the current metadata with snapshot contents,
- emits appropriate `metadata.*` events.

Metadata is generally small, so snapshotting is cheap.

History (undo/redo) integrates at the transaction level; metadata edits are
just normal undoable transactions like other domains.

---

## 6. Relationship to Other Domains

- **Project**  
  Project domain handles load/save, paths, and snapshot lifecycle. Metadata
  augments that with descriptive content and markers.

- **Timebase**  
  Markers and sections are positioned in musical time using the Timebase
  domain. Timebase changes (tempo/signature) do not automatically adjust
  markers; movement semantics are defined in editing guidelines.

- **Tracks/Clips/Lanes**  
  Markers and sections give context to where Clips and Tracks sit. Clips
  themselves may reference section IDs for organisation.

- **Session**  
  Session may expose high-level metadata (title, author, tags) and use it in
  status UIs. Session’s warning system may also refer to metadata issues
  (e.g. missing required tags before export).

- **Composer**  
  Composer may use metadata and markers to build project-level models:
  common section layouts, key usage statistics, tagging conventions, etc.,
  but this is purely advisory and outside the core IPC.

Project Metadata provides the descriptive and structural “glue” around the
project timeline that does not alter the underlying audio/MIDI graph but
helps users and tools understand and navigate it.
