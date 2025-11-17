# Pulse Timebase Domain Specification

This document defines the Timebase domain of the Pulse project protocol.  
It covers commands and events for managing:

- the global tempo map,
- time signatures,
- tempo curves/ramps,
- the relationship between musical time (bars/beats) and absolute time
  (seconds/samples),
- clip-relative timebases.

Pulse owns the authoritative timebase model.  
Signal uses it to schedule audio, automation and cohorts.  
Aura visualises it and provides editing tools (tempo lanes, signature markers).

---

## Contents

- [1. Overview](#1-overview)  
- [2. Timebase Concepts](#2-timebase-concepts)  
  - [2.1 Musical vs Absolute Time](#21-musical-vs-absolute-time)  
  - [2.2 Tempo Map](#22-tempo-map)  
  - [2.3 Time Signatures](#23-time-signatures)  
  - [2.4 Tempo Curves and Ramps](#24-tempo-curves-and-ramps)  
  - [2.5 Global Lanes](#25-global-lanes)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Tempo Markers](#31-tempo-markers)  
  - [3.2 Signature Markers](#32-signature-markers)  
  - [3.3 Range Operations](#33-range-operations)  
  - [3.4 Time Conversion Queries](#34-time-conversion-queries)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Tempo Map Events](#41-tempo-map-events)  
  - [4.2 Signature Map Events](#42-signature-map-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

The Timebase domain defines how Loophole:

- represents time,
- stores tempo and signature changes,
- converts between musical and absolute time,
- informs Signal when tempo changes require anticipative re-scheduling.

Key responsibilities:

- maintain a tempo map over the project timeline,
- maintain a time signature map over the project timeline,
- support linear and curved tempo changes,
- expose conversion utilities for Aura (for rulers, grids, etc.),
- emit events when the timebase changes.

---

## 2. Timebase Concepts

### 2.1 Musical vs Absolute Time

Loophole uses two primary timeaxes:

- **Musical time** – bars, beats, ticks (PPQ),
- **Absolute time** – seconds or samples.

Pulse maintains the mapping between them, based on the tempo map and project
start time.

General principles:

- Transport position is expressed canonically in musical time (bar/beat),
  with an absolute-time equivalent available.
- Automation, clips and lanes express time in whichever domain is most
  appropriate:
  - many musical objects (clips, tempo lanes) are bar/beat-relative,
  - some engine-level structures (buffers, renders) operate in samples.

The Timebase domain centralises conversion logic.

---

### 2.2 Tempo Map

The **tempo map** is a sequence of tempo markers over the musical timeline.

Each tempo marker has:

- `tempoMarkerId`,
- `position` in musical time (bar/beat or PPQ),
- tempo value (e.g. BPM, as a float),
- optional curve/shape to next marker (see 2.4),
- optional metadata (e.g. “tap tempo origin”, “imported from MIDI”).

Between markers, tempo is defined according to the curve/shape associated
with the marker.

The first tempo marker typically resides at project start; if no markers
exist, a default tempo is assumed (e.g. 120 BPM).

---

### 2.3 Time Signatures

The **time signature map** is a sequence of signature markers.

Each signature marker has:

- `signatureMarkerId`,
- `position` in musical time (bar/beat),
- numerator (e.g. 4),
- denominator (e.g. 4),
- optional bar numbering offset.

Time signatures define:

- bar boundaries,
- beat grouping,
- grid/ruler behaviour.

Tempo and time signatures interact but are stored as separate maps.

---

### 2.4 Tempo Curves and Ramps

Tempo changes may be:

- **step** – tempo jumps at the marker position,
- **linear** – tempo interpolates linearly between markers,
- **curve** – tempo follows a more complex curve between markers.

Each tempo marker may specify:

- `shape` (enum: `step`, `linear`, `curve`, etc.),
- optional `shapeParams` for curved transitions (similar conceptually to
  automation point shapes).

The exact interpolation algorithm is an implementation detail. The IPC
schema allows for curved tempo ramps to be added now without constraining
the internal math too tightly.

---

### 2.5 Global Lanes

Tempo and signature information are conceptually global **lanes**:

- **Tempo lane** – visual representation of the tempo map.
- **Signature lane** – visual representation of time signatures.

They are not Tracks in the hybrid Track sense; they are global maps that
apply to the entire project.

The Timebase domain defines the data model for these maps; the Lane/Clip
domains treat them as special global lanes where necessary.

---

## 3. Commands (Aura → Pulse)

Aura issues Timebase commands when the user:

- adds, moves or removes tempo markers,
- changes time signature,
- edits tempo curves,
- performs time stretching of a range,
- requests conversions between time domains.

Pulse updates the project timebase and notifies Signal and Aura via events.

### 3.1 Tempo Markers

#### **`timebase.addTempoMarker`**

Add a tempo marker at a given musical position.

Fields:

- `position` (musical, e.g. bar/beat or PPQ),
- `tempo` (BPM),
- optional `shape` (`step`, `linear`, `curve`),
- optional `shapeParams`.

Pulse:

- inserts the marker into the tempo map,
- returns/announces `tempoMarkerId`,
- emits `timebase.tempoMarkerAdded`.

---

#### **`timebase.updateTempoMarker`**

Update a tempo marker.

Fields may include:

- `tempoMarkerId`,
- new `position`,
- new `tempo`,
- new `shape` / `shapeParams`.

Pulse:

- updates the marker,
- revalidates tempo map ordering,
- emits `timebase.tempoMarkerUpdated`.

---

#### **`timebase.removeTempoMarker`**

Remove a tempo marker.

Fields:

- `tempoMarkerId`.

Pulse updates the map and emits `timebase.tempoMarkerRemoved`.

---

### 3.2 Signature Markers

#### **`timebase.addSignatureMarker`**

Add a time signature marker.

Fields:

- `position` (musical),
- `numerator`,
- `denominator`,
- optional bar-number offset.

Pulse:

- inserts into signature map,
- returns/announces `signatureMarkerId`,
- emits `timebase.signatureMarkerAdded`.

---

#### **`timebase.updateSignatureMarker`**

Update time signature marker fields.

Fields:

- `signatureMarkerId`,
- new `position`, `numerator`, `denominator` or bar offset.

Pulse updates and emits `timebase.signatureMarkerUpdated`.

---

#### **`timebase.removeSignatureMarker`**

Remove a time signature marker.

Fields:

- `signatureMarkerId`.

Pulse emits `timebase.signatureMarkerRemoved`.

---

### 3.3 Range Operations

Range operations support tempo-edit gestures such as “stretch this section” or
“insert measures here”.

#### **`timebase.stretchRange`**

Stretch or compress the musical time between two positions, adjusting tempo
markers accordingly.

Fields:

- `startPosition`,
- `endPosition`,
- `stretchFactor` (e.g. 2.0 = double duration, 0.5 = half).

Pulse adjusts marker positions and tempo values as defined by internal
rules (to be detailed in timebase editing guidelines), then emits summary
events (`timebase.tempoMapChanged`).

---

#### **`timebase.insertTime`**

Insert musical time at a specific position (e.g. insert two bars).

Fields:

- `position`,
- `duration` (musical time, e.g. bars/beats).

Pulse:

- shifts subsequent tempo and signature markers forward,
- emits `timebase.tempoMapChanged` and `timebase.signatureMapChanged`.

---

#### **`timebase.removeTime`**

Remove a span of musical time.

Fields:

- `startPosition`,
- `endPosition`.

Pulse:

- removes or shifts markers falling within/after the range,
- emits appropriate map-changed events.

The behaviour of Clips/Tracks under these operations is defined in the Clip
and Track editing semantics; Timebase is responsible only for its own maps.

---

### 3.4 Time Conversion Queries

#### **`timebase.queryTimeConversion`**

Request conversion between time domains.

Fields:

- `from`:

  - `kind` = `musical` or `absolute`,
  - `value` (musical position or absolute time in seconds/samples),

- `to`:

  - `kind` = `musical` or `absolute`,
  - optional detail (e.g. “seconds” vs “samples”).

Pulse computes and returns a conversion result via a response envelope or
event, depending on IPC design. This is primarily intended for Aura to:

- position visual elements accurately,
- align automation and clip boundaries.

Batch conversion variants may exist, but the spec treats this as a conceptual
operation.

---

## 4. Events (Pulse → Aura)

### 4.1 Tempo Map Events

#### **`timebase.tempoMarkerAdded`**

Tempo marker created.

#### **`timebase.tempoMarkerUpdated`**

Tempo marker updated.

#### **`timebase.tempoMarkerRemoved`**

Tempo marker removed.

#### **`timebase.tempoMapChanged`**

Summary event for bulk operations (stretch, insert/remove time, etc.).  
Aura may re-request relevant sections for precise display.

---

### 4.2 Signature Map Events

#### **`timebase.signatureMarkerAdded`**

Signature marker created.

#### **`timebase.signatureMarkerUpdated`**

Signature marker updated.

#### **`timebase.signatureMarkerRemoved`**

Signature marker removed.

#### **`timebase.signatureMapChanged`**

Summary event indicating larger structural changes to the signature map.

---

## 5. Snapshot Semantics

Snapshots include the full timebase state:

- tempo markers (positions, values, shapes),
- signature markers (positions, numerators, denominators),
- any project-wide default tempo/signature settings.

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

When a snapshot is applied:

- Pulse replaces the tempo and signature maps according to snapshot data,
- emits map-changed events (`tempoMapChanged`, `signatureMapChanged`),
- Signal adjusts scheduling and anticipative rendering accordingly.

Timebase snapshots do not store audio-device-specific timing (e.g. latency);
that belongs to the Hardware I/O domain.

---

## 6. Realtime and Cohort Considerations

Timebase changes can have significant impact on:

- automation timing,
- anticipative buffer scheduling,
- recording alignment.

Therefore:

- Large structural edits (stretch/insert/remove time) are expected to be
  performed when transport is stopped.
- Minor tempo changes during playback are allowed, but may require:

  - cohort rescheduling,
  - anticipative buffer invalidation beyond a certain horizon.

Signal must treat the timebase as:

- read-mostly during playback,
- updated via Pulse with clear event boundaries,
- safe to sample for scheduling decisions without heavy locking.

Exact policies for tempo changes while playing are detailed in the cohort and
transport semantics documents.

---

## 7. Relationship to Other Domains

The Timebase domain interacts with:

- **Transport**  
  Transport positions, looping and locating use the timebase for conversion
  between musical and absolute time.

- **Clips and Lanes**  
  Clip start/end positions in musical time depend on the tempo/signature map.
  Range operations (insert/remove time) require coordination with clip editing
  rules.

- **Automation**  
  Automation envelopes with musical positions rely on the timebase for
  scheduling parameter changes. Conversion queries may be needed when
  rendering or bouncing.

- **Processing Cohorts**  
  Anticipative rendering uses the tempo map to determine how far ahead in
  time to render and to align render blocks with transport movement.

- **Hardware I/O**  
  Sample rate from Hardware I/O affects the absolute-time axis (samples ↔
  seconds), but not the musical timebase directly.

The Timebase domain provides a coherent, centralised representation of tempo
and signatures that all timing-sensitive subsystems rely upon.
