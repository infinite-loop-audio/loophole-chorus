# Pulse Rendering Domain Specification

This document defines the Rendering domain of the Pulse project protocol.  
It covers all operations where the engine produces **offline or non-live
rendered media**, including:

- Track Freeze / Unfreeze,
- Clip / Range Bounce,
- “Render In Place” (replace region with audio),
- Full Mixdown exports,
- Background rendering jobs (with progress events).

Rendering is conceptually similar to Recording, but inverted:

- Recording captures *incoming* signals,
- Rendering generates *output* signals into media.

Pulse coordinates rendering requests, instructs Signal to execute renders,  
and registers results in the Media domain.

Aura triggers renders and monitors progress.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Rendering Concepts](#2-rendering-concepts)  
  - [2.1 Render Targets](#21-render-targets)  
  - [2.2 Render Modes](#22-render-modes)  
  - [2.3 Freeze Semantics](#23-freeze-semantics)  
  - [2.4 Rendering Range](#24-rendering-range)  
  - [2.5 Rendering Cohorts](#25-rendering-cohorts)  
  - [2.6 Result Media and Registration](#26-result-media-and-registration)  
  - [2.7 Background Task Integration](#27-background-task-integration)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Freeze Operations](#31-freeze-operations)  
  - [3.2 Bounce / Render Range](#32-bounce--render-range)  
  - [3.3 Render In Place](#33-render-in-place)  
  - [3.4 Full Mixdown / Export](#34-full-mixdown--export)  
  - [3.5 Job Management](#35-job-management)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Job Lifecycle Events](#41-job-lifecycle-events)  
  - [4.2 Result Events](#42-result-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

Rendering is a general-purpose engine operation that produces media files
from the internal audio graph.

Pulse owns:

- job orchestration and job state,
- mapping render outputs to media IDs,
- freeze/unfreeze bookkeeping,
- integration with the Media domain.

Signal owns:

- the actual audio rendering,
- feeding results back to Pulse.

Aura triggers rendering and displays job progress and results.

---

## 2. Rendering Concepts

### 2.1 Render Targets

A render target defines **what** is being rendered. Common target types:

- **Track** – render the entire output of a Track (all Lanes mixed).
- **Lane** – render a single Lane (audio or instrument Lane).
- **Node** – render the output of a specific Node chain.
- **Channel** – render a bus or group output.
- **Master** – render the full mix (for export).
- **Selection / Range** – render a time range of the whole graph or subset.

Targets are expressed using domain-specific IDs:

- `trackId`
- `laneId`
- `nodeId`
- `channelId`

Pulse transforms these into a concrete engine render job.

---

### 2.2 Render Modes

Typical rendering modes:

- **offline** – fast as possible, non-realtime (default for bounce).
- **realtime** – match hardware playback timing (rare; only used in special
  cases).
- **freeze** – freeze a Track/Lane to temporary media for CPU savings.
- **replace** – render and overwrite/replace Clip or Lane content.
- **preview** – render short segments for auditioning.

The IPC domain does not hard-code the exact mode semantics; these are
recorded as hints in a job definition.

---

### 2.3 Freeze Semantics

A **freeze** operation:

1. Renders the output of a Track or Lane.
2. Produces one or more Media Items.
3. Marks the Track/Lane/Node as “frozen”.
4. Disables/frees CPU load from underlying Nodes.
5. Allows reversible restoration via “unfreeze”.

Freeze metadata stored in Pulse:

- which Nodes were bypassed,
- the Media Items created,
- the time ranges rendered,
- configuration needed to reverse the freeze.

Unfreeze:

- removes or hides frozen media,
- restores Node graph to its original state.

Signal never knows about “freeze” conceptually; it just renders the graph.

---

### 2.4 Rendering Range

A render may specify:

- **musical positions** (bar/beat range),
- **absolute time** (seconds),
- **entire Clip**, **entire Track**, or **entire project**,
- optional **tail** (fadeout or decay time for FX).

Pulse validates and normalises ranges before sending to Signal.

---

### 2.5 Rendering Cohorts

Rendering uses cohorts differently:

- offline renders may temporarily convert Nodes to **offline cohorts**,
- live-cohort Nodes that require real-time behaviour may need to be replaced
  with offline-safe equivalents,
- anticipative cohorts may expand render horizon aggressively,
- rendering may run with a modified automation schedule.

Pulse configures render cohorts for each job.

---

### 2.6 Result Media and Registration

All rendering outputs become Media Items.

The steps:

1. Pulse asks Media domain for a recording target (`media.reserveRecordingTarget`).
2. Signal renders into the allocated file.
3. Pulse finalises the render (`media.finaliseRecording`).
4. Pulse associates the render result with the job:

   - for freeze → attach to Track/Lane,
   - for bounce-in-place → replace Clip content,
   - for export → expose file path.

Aura receives `rendering.jobCompleted` with the relevant media IDs.

---

### 2.7 Background Task Integration

Rendering jobs are long-running background tasks.

- Session domain reports high-level “background tasks active”.
- Engine Diagnostics may expose deeper performance details.
- Media domain handles final file registration and analysis.

---

## 3. Commands (Aura → Pulse)

### 3.1 Freeze Operations

#### **`rendering.freezeTrack`**

Freeze an entire Track.

Fields:

- `trackId`,
- optional `mode` (`full`, `preFX`, `postFX` depending on design),
- optional `range` (musical/absolute; defaults to full Track length).

Pulse:

- validates Track,
- creates a freeze job,
- signals Signal to render,
- job lifecycle reported via events.

---

#### **`rendering.unfreezeTrack`**

Reverse a freeze.

Fields:

- `trackId`.

Pulse restores the original Node graph and removes frozen media references.

---

#### **`rendering.freezeLane`**

Freeze a single Lane.

Fields:

- `laneId`,
- optional mode/range as above.

---

### 3.2 Bounce / Render Range

#### **`rendering.renderRange`**

General-purpose bounce/range render.

Fields:

- `target`:

  - `trackId` OR  
  - `laneId` OR  
  - `channelId` OR  
  - `nodeId` OR  
  - `selection` structure,

- `range`:

  - musical or absolute time range,

- `mode` (`offline` | `realtime` | `preview`),
- optional `tailDuration`.

Pulse returns a `renderJobId`.

---

### 3.3 Render In Place

#### **`rendering.renderInPlace`**

Render the selected Clips/Lanes and replace them with rendered audio.

Fields:

- `targets` (one or more Clip or Lane identifiers),
- `range` (optional; defaults to union of Clip ranges),
- optional settings (tail, preservation of fades/envelopes).

After render:

- Pulse updates Clip/Lane media references,
- original rendered content may be preserved in Media Pool for undo.

---

### 3.4 Full Mixdown / Export

#### **`rendering.exportMixdown`**

Render the master bus or other export target to a media file.

Fields:

- `range` (musical or absolute),
- `exportSettings`:

  - sample rate,
  - bit depth / float,
  - channel format (mono/stereo/multichannel),
  - file format,
  - dither,
  - normalisation,
  - metadata tags.

Pulse coordinates file allocation and job execution.

Aura receives progress and final file details.

---

### 3.5 Job Management

#### **`rendering.cancelJob`**

Cancel a running job.

Fields:

- `renderJobId`.

Signal stops the render when safe; Pulse finalises job with a cancellation
result.

---

#### **`rendering.queryJob`**

Request job status.

Fields:

- `renderJobId`.

Pulse responds with a minimal job-status envelope.

---

## 4. Events (Pulse → Aura)

### 4.1 Job Lifecycle Events

#### **`rendering.jobStarted`**

Fields:

- `renderJobId`,
- `target` summary,
- optional human-readable description.

---

#### **`rendering.jobProgress`**

Periodic progress update.

Fields:

- `renderJobId`,
- `progress` (0–1),
- optional details (current position/time, stage).

---

#### **`rendering.jobCompleted`**

Render finished successfully.

Fields:

- `renderJobId`,
- list of resulting `mediaId`s,
- optional target-specific fields:

  - for freeze → track/lane frozen state update,
  - for replace → updated Clip IDs.

---

#### **`rendering.jobFailed`**

Render encountered an error.

Fields:

- `renderJobId`,
- error description,
- recovery suggestions (if possible).

---

#### **`rendering.jobCancelled`**

Job cancelled by user or host.

Fields:

- `renderJobId`.

---

## 5. Snapshot Semantics

Project snapshots **do not** include:

- active render jobs,
- temporary freeze artifacts still being written.

Snapshots **do** include:

- frozen state of Tracks/Lanes at snapshot time,
- references to frozen media.

When reversing a snapshot (undo):

- Pulse restores frozen states according to snapshot data,
- does not restart or retroactively re-run renders.

---

## 6. Realtime and Cohort Considerations

Rendering is always **non-realtime** unless explicitly requested.

Signal performs:

- bypass of hardware-dependent Nodes,
- execution in offline-safe cohorts,
- deterministic stepping through automation and timebase.

Cohorts may be temporarily reconfigured:

- freeze operations collapse entire Track graphs into single offline nodes,
- live nodes may be overridden or replaced with offline fallbacks.

Latency and hardware buffer sizes do not apply to offline renders.

---

## 7. Relationship to Other Domains

The Rendering domain interacts with:

- **Media**  
  All render outputs become Media Items.

- **Recording**  
  Rendering mirrors recording behaviour; both share reserve/finalise patterns.

- **Nodes and Channels**  
  Renders depend on the Node graph; freeze collapses Nodes into media.

- **Routing**  
  Render targets map to internal routing objects.

- **Timebase**  
  Time ranges and automation alignment depend on tempo maps.

- **Session**  
  Rendering jobs appear as background tasks in Session.

- **Engine Diagnostics**  
  Long-running renders may report CPU usage and cohort behaviour via diagnostics.

Rendering provides the backbone for bounce/freeze workflows and exporting
audio from Loophole.
