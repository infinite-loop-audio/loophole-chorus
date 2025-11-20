# Pulse Engine Diagnostics Domain Specification

This document defines the Engine Diagnostics domain of the Pulse project
protocol.

It covers read-only, status-oriented commands and events for:

- engine run state,
- processing cohort status (live vs anticipative),
- CPU load and performance metrics,
- render horizon depth,
- underrun/overload indicators (beyond raw hardware I/O),
- plugin failure / isolation state,
- safe-mode and degraded operation flags.

Pulse aggregates diagnostic information from Signal and exposes a stable,
UI-friendly view to Aura. Aura uses this to render status indicators,
debug/inspector panels and development tools.

Diagnostics are **non-authoritative** for audio behaviour – changing them
does not change engine state – but they must accurately reflect engine state.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Diagnostics Concepts](#2-diagnostics-concepts)  
  - [2.1 Engine State](#21-engine-state)  
  - [2.2 Cohorts and Render Horizon](#22-cohorts-and-render-horizon)  
  - [2.3 CPU and Performance Metrics](#23-cpu-and-performance-metrics)  
  - [2.4 Plugin Health and Isolation](#24-plugin-health-and-isolation)  
  - [2.5 Severity Levels](#25-severity-levels)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Snapshot Requests](#31-snapshot-requests)  
  - [3.2 Subscription Control](#32-subscription-control)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Engine State Events](#41-engine-state-events)  
  - [4.2 Cohort and Scheduling Events](#42-cohort-and-scheduling-events)  
  - [4.3 Performance and Load Events](#43-performance-and-load-events)  
  - [4.4 Plugin Health Events](#44-plugin-health-events)  
  - [4.5 Generic Diagnostic Events](#45-generic-diagnostic-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Safety Considerations](#6-realtime-and-safety-considerations)  
- [7. Relationship to Other Domains](#7-relationship-to-other-domains)

---

## 1. Overview

The Engine Diagnostics domain provides a structured way to query and monitor:

- whether the engine is running,
- how cohorts are arranged (live vs anticipative),
- how far ahead anticipative rendering is progressing,
- how busy the CPU is (both overall and per-cohort),
- where buffer underruns/overloads are occurring,
- whether plugins have crashed or been quarantined,
- whether the engine has entered a “safe mode”.

Diagnostics are primarily intended for:

- status bars and warning indicators,
- inspector/debug panels,
- development tooling,
- advanced user-facing analysis tools.

They are advisory and must not be used to drive core audio logic.

---

## 2. Diagnostics Concepts

### 2.1 Engine State

Engine state describes the high-level running status of Signal, for example:

- `stopped` – engine initialised but not processing audio,
- `running` – engine processing audio in realtime or anticipative mode,
- `suspended` – engine paused due to error or device loss,
- `safeMode` – engine running with degraded capabilities (e.g. bypassing
  problematic plugins or disabling anticipative processing).

Diagnostics expose this as read-only state, updated as Signal transitions.

---

### 2.2 Cohorts and Render Horizon

Loophole uses processing cohorts, conceptually:

- **live cohort** – nodes that must respond with minimal latency,
- **anticipative cohorts** – nodes that can be rendered ahead of time.

Diagnostics expose:

- the set of cohorts known to Signal,
- which nodes (or Channels) belong to which cohort,
- the **render horizon** for anticipative cohorts:

  - how far ahead (in musical/absolute time) they have rendered relative to
    the current transport position.

This allows Aura to visualise:

- which parts of the graph are being processed in advance,
- when anticipative buffers are being invalidated or rebuilt.

---

### 2.3 CPU and Performance Metrics

Diagnostics expose CPU-related metrics such as:

- overall audio callback CPU utilisation (percentage),
- per-cohort CPU utilisation estimates,
- per-node or per-channel hotspots (coarse-grained; detailed profiling is a
  separate concern),
- recent buffer underrun/overrun counts (beyond raw hardware glitch signals).

These are intended for:

- user guidance (e.g. “CPU is overloaded”),
- developer debugging,
- non-real-time visualisation.

Exact sampling strategy and smoothing are implementation details; the IPC
schema allows periodic summary updates.

---

### 2.4 Plugin Health and Isolation

Diagnostics report plugin-related issues, e.g.:

- plugin crash detected,
- plugin instance quarantined or disabled,
- plugin causing repeated underruns,
- plugin running in an isolated process (if implemented).

This includes:

- identifiers for affected Nodes and Channels,
- plugin metadata (name, vendor),
- a severity level and suggested action (e.g. “bypassed”, “reloaded”).

The diagnostic system does not define the isolation mechanism; it only
reports its results.

---

### 2.5 Severity Levels

Diagnostic messages may carry a severity:

- `info` – informative, no action required,
- `warning` – something may be suboptimal,
- `error` – something has failed but the engine continues,
- `critical` – unrecoverable error or severe degradation.

Aura can filter and present these appropriately.

---

## 3. Commands (Aura → Pulse)

### 3.1 Snapshot Requests

#### **`engineDiagnostics.requestSnapshot`**

Request a one-off snapshot of engine diagnostics.

Fields (all optional filters):

- `includeEngineState` (bool),
- `includeCohorts` (bool),
- `includePerformance` (bool),
- `includePlugins` (bool).

Pulse queries Signal and responds via `engineDiagnostics.snapshotUpdated`.

---

### 3.2 Subscription Control

#### **`engineDiagnostics.subscribe`**

Subscribe to ongoing diagnostic updates.

Fields (optional):

- frequency hint (e.g. updates per second),
- which categories to include (engine state, cohorts, performance, plugins).

#### **`engineDiagnostics.unsubscribe`**

Stop receiving diagnostic updates.

This is useful to reduce overhead when diagnostic UI is not visible.

---

## 4. Events (Pulse → Aura)

### 4.1 Engine State Events

#### **`engineDiagnostics.engineStateChanged`**

Emitted when high-level engine state changes.

Includes:

- `state` (`stopped`, `running`, `suspended`, `safeMode`, etc.),
- optional reason or explanation string,
- optional timestamp.

Aura may use this for:

- status bar indicators,
- warning dialogs when entering safe mode or suspended state.

---

### 4.2 Cohort and Scheduling Events

#### **`engineDiagnostics.cohortsUpdated`**

Snapshot of cohort configuration.

Includes:

- a list of cohorts, each with:

  - cohort identifier,
  - type (`live`, `anticipative`, or other future types),
  - list of associated Channels/Nodes (by ID) or summary counts.

---

#### **`engineDiagnostics.renderHorizonUpdated`**

Information about anticipative rendering progress.

Fields:

- per-cohort horizon:

  - `cohortId`,
  - horizon position in musical and/or absolute time relative to current
    transport.

This allows visualisation of how far ahead audio is pre-rendered.

---

### 4.3 Performance and Load Events

#### **`engineDiagnostics.performanceUpdated`**

Periodic summary of performance metrics.

May include:

- overall CPU usage (percentage),
- audio callback load estimates,
- per-cohort CPU approximations,
- recent underrun/overrun counts,
- queue or buffer utilisation metrics.

Aura uses this to render CPU meters and performance indicators.

---

### 4.4 Plugin Health Events

#### **`engineDiagnostics.pluginIssueDetected`**

Reports an issue with a plugin instance.

Includes:

- `nodeId` / `channelId` / `trackId`,
- plugin metadata (name, vendor, version),
- severity level,
- issue type (e.g. `crash`, `timeout`, `repeatedUnderruns`, `initialisationFailed`),
- action taken (e.g. `bypassed`, `unloaded`, `isolated`),
- suggested user-visible message.

---

#### **`engineDiagnostics.pluginIssueCleared`**

Reports that a previously flagged issue is no longer present.

Includes:

- `nodeId`,
- issue ID or type.

---

### 4.5 Generic Diagnostic Events

#### **`engineDiagnostics.snapshotUpdated`** (event — domain: engineDiagnostics)

Emitted in response to `engineDiagnostics.requestSnapshot` command. This is an event, not a direct response envelope.

Contains a structured snapshot:

- engine state,
- cohorts and horizons,
- performance metrics,
- plugin health summaries,

according to the requested categories.

---

#### **`engineDiagnostics.message`**

Generic diagnostic message suitable for logging or advanced UIs.

Includes:

- severity (`info`, `warning`, `error`, `critical`),
- source (e.g. `engine`, `cohort`, `plugin`, `hardware`),
- message text,
- optional structured data (key-value metadata),
- optional correlation ID.

Aura may display these in a diagnostics console.

---

## 5. Snapshot Semantics

Project snapshots (as in project state, undo/redo) **do not include**
engine diagnostics.

The Engine Diagnostics domain refers to:

- transient measurements,
- runtime health,
- environment-dependent state.

However, diagnostics may be recorded in:

- debug logs,
- performance recordings,
- profiling sessions.

Snapshot semantics here refer only to `engineDiagnostics.requestSnapshot`,
which is a **diagnostic snapshot**, not a project snapshot.

---

## 6. Realtime and Safety Considerations

Diagnostics must never compromise realtime safety:

- all heavy work (aggregation, formatting) happens off the audio thread,
- collection from Signal should use lock-free or low-overhead mechanisms,
- sampling frequency must be bounded (and controllable via subscription).

In case of severe overload:

- diagnostic reporting may be throttled or suspended to prioritise audio.

Engine Diagnostics are strictly **observational**; no command in this domain
may directly control engine behaviour (start/stop/modify graph). That
belongs to other domains (Transport, Project, Routing, etc.).

---

## 7. Relationship to Other Domains

The Engine Diagnostics domain relates to:

- **Hardware I/O**  
  Hardware errors and xruns may be surfaced both via Hardware I/O events and
  Engine Diagnostics. Hardware I/O focuses on device-level issues; Engine
  Diagnostics focuses on their impact on engine performance.

- **Processing Cohorts**  
  Processing cohort (PC) definitions and behaviour are described in architecture docs; this
  domain reports their runtime status and performance.

- **Automation and Gesture**  
  Heavy automation or gesture load may show up as CPU spikes; diagnostics
  expose this correlation but do not change control logic.

- **Plugin and Node domains**  
  Plugin/node issues discovered at runtime are reported here; plugin loading
  and configuration are defined elsewhere.

Engine Diagnostics provides the **visibility layer** over the runtime
behaviour of Signal, without altering its operation.
