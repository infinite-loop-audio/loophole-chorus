# Pulse Recording Domain Specification

This document defines the Recording domain of the Pulse project protocol.  
It covers commands and events for configuring and controlling **recording
behaviour**, including:

- arming Tracks/Lanes for recording,
- selecting input sources (hardware, busses, internal),
- monitor modes (input/tape/auto),
- record modes (takes, overdub, replace, layered),
- punch-in/out ranges and pre-roll behaviour,
- reporting recorded material back to Aura.

Pulse coordinates recording configuration and state.  
Signal performs the actual capture of audio/MIDI and writes media to disk.  
Aura presents controls and comping interfaces.

Transport start/stop and record flags are defined in the Transport domain;  
Recording focuses on *what* gets recorded *where* and *how*.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Recording Concepts](#2-recording-concepts)  
  - [2.1 Arms and Targets](#21-arms-and-targets)  
  - [2.2 Input Sources](#22-input-sources)  
  - [2.3 Monitor Modes](#23-monitor-modes)  
  - [2.4 Record Modes and Takes](#24-record-modes-and-takes)  
  - [2.5 Punch and Pre-Roll](#25-punch-and-pre-roll)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Arming and Monitoring](#31-arming-and-monitoring)  
  - [3.2 Input Source Configuration](#32-input-source-configuration)  
  - [3.3 Record Modes and Takes](#33-record-modes-and-takes)  
  - [3.4 Punch and Pre-Roll](#34-punch-and-pre-roll)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Arming and Monitoring Events](#41-arming-and-monitoring-events)  
  - [4.2 Input Source Events](#42-input-source-events)  
  - [4.3 Record Mode and Take Events](#43-record-mode-and-take-events)  
  - [4.4 Recording Results](#44-recording-results)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

Recording in Loophole is built on several layers:

- **Hardware I/O** – exposes physical inputs,
- **Routing/Channels** – define where input signals enter the internal graph,
- **Tracks/Lanes** – define where recorded material is stored and how it is
  organised,
- **Clips** – represent recorded segments as timeline objects,
- **Timebase** – provides tempo and musical position for recording alignment,
- **Transport** – controls when recording starts/stops.

The Recording domain defines:

- which Tracks/Lanes are armed,
- which inputs feed each armed target,
- monitoring behaviour during recording and playback,
- how recordings are captured as takes/clips,
- how results are reported back to Aura.

---

## 2. Recording Concepts

### 2.1 Arms and Targets

Recording targets are:

- **Tracks** – high-level arm state, used for UI and default behaviour,
- **Lanes** – concrete destinations for recorded material (audio/MIDI).

Arming may be:

- **Track-level** – “record-enable this Track” (fan out to compatible Lanes),
- **Lane-level** – “explicitly record-enable this Lane” (fine-grained).

Pulse maintains:

- a record-enabled flag per Track,
- record-enabled flags per Lane,
- any derived relationships between them (e.g. Track arm implies default
  Lane arms, depending on Lane types/settings).

Signal relies on Pulse to tell it which Lanes are active recording targets.

---

### 2.2 Input Sources

An input source describes where the signal comes from:

- **Hardware input**:

  - via Hardware I/O roles/aliases,
  - e.g. `role: "input.vocal"`, or direct `(deviceId, channelIndices)`.

- **Bus/Channel input**:

  - e.g. recording from a submix, FX return or internal bus.

- **Instrument/MIDI input**:

  - hardware MIDI ports,
  - virtual MIDI sources,
  - other internal generator Nodes.

The Recording domain specifies *which source* is attached to each recording
target, using abstractions defined in Hardware I/O and Routing domains.

---

### 2.3 Monitor Modes

Monitor mode controls what is heard on an armed Track/Lane:

- **input** – monitor live input whenever armed,
- **tape** – monitor input only when recording; monitor playback otherwise,
- **auto** – behaviour that depends on transport/record state (host-defined,
  but typically: input when stopped or recording, playback on normal play),
- **off** – monitor disabled (no input monitoring).

Monitor mode is defined per Track and/or per Lane (depending on use case).
Pulse communicates the desired mode to Signal so that it can route live
monitoring correctly.

---

### 2.4 Record Modes and Takes

Record mode defines how new material is written:

- **overwrite** – existing material in the record range is replaced,
- **layered** – new recordings stack on top of existing material,
- **takes** – new recordings create new “takes” within a take lane/comping
  structure.

Take management semantics (comping, take lanes) are defined in the Clip/Lane
architecture, but Recording must:

- decide whether a new take lane is created or an existing lane is reused,
- generate identifiers for new takes/material segments,
- report them back to Aura so that comping UI can be updated.

---

### 2.5 Punch and Pre-Roll

Punch and pre-roll define how recording is constrained in time:

- **Punch range** – a time range where recording is actually written,
  typically defined in musical time:

  - `punchInPosition`,
  - `punchOutPosition`.

- **Pre-roll** – playback prior to the punch-in point, for context, while
  arming the engine ready to record.

Pre-roll may be:

- global (project preference),
- or per-record pass (user-specified for a given record start).

Transport domain controls transport start/stop; Recording domain specifies
punch ranges and pre-roll preferences that affect how Signal treats incoming
data before/after punch boundaries.

---

## 3. Commands (Aura → Pulse)

### 3.1 Arming and Monitoring

#### **`recording.setTrackArmed`**

Set the record-armed state for a Track.

Fields:

- `trackId`,
- `armed` (boolean).

Pulse:

- updates Track arm state,
- may propagate to suitable Lanes (depending on Lane types and settings),
- emits `recording.trackArmedChanged`.

---

#### **`recording.setLaneArmed`**

Set the record-armed state for a specific Lane.

Fields:

- `laneId`,
- `armed` (boolean).

Used for fine-grained control (e.g. arming only a specific audio Lane for
multi-Lane Tracks).

---

#### **`recording.setTrackMonitorMode`**

Set monitor mode for a Track.

Fields:

- `trackId`,
- `mode` (`input`, `tape`, `auto`, `off`).

---

#### **`recording.setLaneMonitorMode`**

Set monitor mode for a specific Lane.

Fields:

- `laneId`,
- `mode` as above.

Pulse resolves effective monitoring behaviour and instructs Signal
accordingly via internal protocols.

---

### 3.2 Input Source Configuration

#### **`recording.setLaneInputSource`**

Set the input source for a Lane.

Fields may include:

- `laneId`,
- `source` descriptor, which may be:

  - hardware role (`role: "input.vocal"`),
  - explicit hardware mapping (`deviceId`, `channelIndices`),
  - internal bus/Channel reference (`channelId`),
  - MIDI source (MIDI device/port ID, virtual bus Id).

The precise schema for `source` should align with Hardware I/O and Routing
domain schemas; Recording does not re-define those structures, only uses
them.

Pulse validates source compatibility with Lane type (audio vs MIDI) and
updates Signal routing as needed.

---

#### **`recording.clearLaneInputSource`**

Remove or reset the input source for a Lane.

Fields:

- `laneId`.

---

### 3.3 Record Modes and Takes

#### **`recording.setTrackRecordMode`**

Set record mode for a Track.

Fields:

- `trackId`,
- `mode` (e.g. `overwrite`, `layered`, `takes`).

---

#### **`recording.setLaneRecordMode`**

Optionally override record mode on a per-Lane basis.

Fields:

- `laneId`,
- `mode`.

This allows advanced workflows where different Lanes on the same Track use
different record modes.

---

### 3.4 Punch and Pre-Roll

#### **`recording.setPunchRange`**

Define a punch-in/out range.

Fields:

- `punchEnabled` (bool),
- `punchInPosition` (musical time),
- `punchOutPosition` (musical time).

Transport is expected to honour this range when in record mode; Signal may
buffer input outside the punch range as needed.

---

#### **`recording.setPreRoll`**

Set pre-roll behaviour.

Fields:

- `enabled` (bool),
- `duration` (musical or absolute time),
- optional `countIn` flag (metronome click before recording).

---

## 4. Events (Pulse → Aura)

### 4.1 Arming and Monitoring Events

#### **`recording.trackArmedChanged`**

Track arm state changed.

Fields:

- `trackId`,
- `armed`.

---

#### **`recording.laneArmedChanged`**

Lane arm state changed.

Fields:

- `laneId`,
- `armed`.

---

#### **`recording.trackMonitorModeChanged`**

Track monitor mode changed.

---

#### **`recording.laneMonitorModeChanged`**

Lane monitor mode changed.

Aura uses these to update UI indicators and control surfaces.

---

### 4.2 Input Source Events

#### **`recording.laneInputSourceChanged`**

Input source for a Lane changed.

Fields:

- `laneId`,
- new `source` descriptor.

Useful when sources are changed externally (e.g. via control surfaces or
scripts).

---

### 4.3 Record Mode and Take Events

#### **`recording.trackRecordModeChanged`**

Track record mode changed.

#### **`recording.laneRecordModeChanged`**

Lane-level override changed.

---

### 4.4 Recording Results

After a recording pass (i.e. Transport enters and then leaves “recording”
state), Pulse must inform Aura what material has been created.

#### **`recording.passCompleted`**

Summary event for a record pass.

Fields may include:

- pass identifier,
- time range covered (start/end positions),
- list of **recorded items**, each with:

  - `laneId`,
  - `clipId` or equivalent identifier in the Clip domain,
  - media reference (audio file ID, MIDI sequence ID),
  - take identifier (if in `takes` mode),
  - metadata (e.g. number of dropped samples if any).

The exact mapping to Clip and media models is governed by their respective
architecture and IPC specs. The Recording domain is responsible for signalling
that new material exists and where.

Additional events (e.g. per-Lane “recordingStarted” / “recordingStopped”) may
be added if required for finer-grained UI animation, but `recording.passCompleted`
provides a high-level correlation point.

---

## 5. Snapshot Semantics

Project snapshots include:

- arm states (Tracks/Lanes),
- monitor modes,
- record modes,
- punch ranges and pre-roll settings.

Snapshots do **not** include:

- transient record passes in progress,
- partial data not yet committed to Clips/media.

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

When applying a snapshot:

- Recording configuration is restored,
- no attempt is made to retroactively alter past recordings.

Undo/redo of recording passes may be modelled via Clip/media history rather
than Recording configuration itself; Recording snapshots focus on configuration.

---

## 6. Realtime and Cohort Considerations

Recording interacts closely with realtime and processing cohort (PC) logic:

- Armed Lanes typically belong to the **live cohort** or must at least have
  live monitoring paths, even if their downstream processing can be
  anticipative.
- Input monitoring requires:

  - low latency,
  - careful synchronisation with hardware latency (from Hardware I/O).

- Punch behaviour may require:

  - early arming and buffering before punch-in,
  - precise cut points at punch boundaries,
  - coordination with Timebase for alignment.

Signal must:

- avoid disk I/O blocking the audio thread (use background threads and
  buffers),
- ensure recording uses stable time references (Timebase, Hardware latency).

Pulse ensures that:

- configuration is coherent before a record pass starts,
- changes to arm state or input sources during recording are handled
  according to host policy (either applied immediately or deferred).

---

## 7. Relationship to Other Domains

Recording depends on and interacts with:

- **Transport**  
  Transport commands control when recording is active (record on/off).  
  Recording domain defines what happens during that period.

- **Hardware I/O and Routing**  
  Recording input sources rely on Hardware I/O roles/aliases and Routing
  mappings for physical inputs and internal busses.

- **Tracks, Lanes and Clips**  
  Recording targets are Lanes; results become Clips with associated media.

- **Timebase**  
  Punch ranges and recorded clip positions are expressed in musical time and
  must respect tempo and signature changes.

- **Automation and Gesture**  
  Gesture streams may be recorded into automation while recording audio/MIDI;
  Recording domain coexists with that behaviour but does not govern it.

The Recording domain provides the configuration and reporting layer that ties
together engine capture (Signal) and editing/comping UI (Aura) via Pulse.
