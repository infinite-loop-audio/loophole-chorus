# Signal IPC – Diagnostics Domain

This document defines the **Diagnostics** IPC domain between Pulse and Signal.

The Diagnostics domain is responsible for:

- reporting **engine health** (underruns, overloads, callback timing),
- reporting **performance metrics** (CPU load per engine, per cohort, per node),
- exposing **latency and buffering** information,
- providing **debug snapshots** of the engine state for tooling and support.

Pulse owns:

- how diagnostics are surfaced in Aura (meters, warning banners, performance
  inspector),
- correlation with project model (Tracks, Channels, Nodes, Clips),
- persistence of diagnostic logs when needed (e.g. for bug reports).

Signal owns:

- accurate measurement of engine runtime behaviour,
- mapping diagnostics to the engine’s notion of Channels, Nodes, and cohorts,
- ensuring diagnostics collection is lightweight and does not destabilise the
  audio callback.

Aura never talks directly to Signal; all diagnostics messages flow **Signal → Pulse**,
with optional control commands **Pulse → Signal** (e.g. toggling detail levels).

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Diagnostics domain:

- monitors audio callback timing:
  - buffer processing durations,
  - underruns / dropouts,
  - scheduling jitter,
- monitors CPU and memory usage at appropriate granularity:
  - engine overall,
  - per-cohort (real-time vs anticipative),
  - per-channel / per-node where feasible,
- exposes latency and buffering information:
  - effective IO latency (input + output),
  - internal buffering for anticipative rendering,
- provides optional debug snapshots:
  - summarised views of the engine graph and its performance,
  - error logs relevant to Signal.

### 1.2 Non-Goals

The Diagnostics domain does **not**:

- modify engine behaviour directly (no “tuning” commands here),
- expose low-level implementation details of the engine’s internals beyond what
  is needed for meaningful UX,
- replace profiling or logging tools used during development (`perf`,
  `Instruments`, etc.),
- report every individual callback’s timing; it focuses on aggregated metrics.

---

## 2. Metrics & Sampling Model

Diagnostics should be **sampled and aggregated**, not spammy:

- Metrics are computed over windows (e.g. 0.5–2 seconds), not per-buffer.
- Events are emitted:
  - on thresholds being crossed,
  - at a regular low rate (e.g. once per second) for summary stats.

### 2.1 Time Windows

Diagnostics windows (configurable, but typical):

- `windowDurationMs`: 1000–2000 ms for summary stats,
- separate tracking for:
  - short-term spikes (e.g. last few buffers),
  - medium-term averages.

### 2.2 Levels of Detail

Signal can operate with **diagnostic levels**:

- `off` – minimal/no reporting (only critical failures),
- `basic` – high-level engine stats and underrun events,
- `detailed` – cohort and channel/node-level stats where feasible,
- `debug` – more verbose info intended for development and support, not for
  normal use.

Pulse can control this (within reason) via `diagnostics.setLevel`.

---

## 3. Commands (Pulse → Signal)

Diagnostics is mostly event-driven, but there are a few control/query commands.

### 3.1 `diagnostics.setLevel`

Set the diagnostics detail level.

**Request**

```json
{
  "command": "diagnostics.setLevel",
  "level": "basic" // "off" | "basic" | "detailed" | "debug"
}
```

Semantics:

- Signal should adjust its internal tracking accordingly:
  - `off`: only fatal errors/critical problems,
  - `basic`: engine-level CPU, underruns, buffer stats,
  - `detailed`: per-cohort and per-channel/node stats,
  - `debug`: may include additional trace/logging; may have a performance cost.
- Pulse should default to `basic` in normal operation.

---

### 3.2 `diagnostics.getSummary`

Request a one-shot diagnostics summary.

**Request**

```json
{
  "command": "diagnostics.getSummary"
}
```

**Response**

```json
{
  "replyTo": "diagnostics.getSummary",
  "engine": {
    "cpuLoadPercent": 15.3,
    "xruns": 0,
    "bufferUnderruns": 0,
    "averageCallbackMs": 0.7,
    "maxCallbackMs": 1.1
  },
  "cohorts": [
    {
      "cohortId": "cohort:rt:master",
      "cpuLoadPercent": 8.1
    },
    {
      "cohortId": "cohort:anticipative:mix",
      "cpuLoadPercent": 11.4
    }
  ],
  "latency": {
    "inputMs": 5.0,
    "outputMs": 5.0,
    "engineMs": 0.5
  }
}
```

Semantics:

- Values are aggregated over an implementation-defined recent period.
- Useful for:
  - performance panels,
  - quick “is my engine healthy?” checks,
  - bug report snapshots.

---

### 3.3 `diagnostics.getNodeStats`

Request diagnostics for specific Channels/Nodes (only meaningful at `detailed`
or `debug` levels).

**Request**

```json
{
  "command": "diagnostics.getNodeStats",
  "channelIds": ["ch:track:42", "ch:bus:drums"],
  "nodeIds": ["node:track:42:plugin:1"]
}
```

**Response**

```json
{
  "replyTo": "diagnostics.getNodeStats",
  "channels": [
    {
      "channelId": "ch:track:42",
      "cpuLoadPercent": 4.2,
      "xruns": 0
    }
  ],
  "nodes": [
    {
      "nodeId": "node:track:42:plugin:1",
      "cpuLoadPercent": 2.8,
      "xruns": 0
    }
  ]
}
```

Semantics:

- Not guaranteed to be cheap; Pulse should use sparingly, e.g. on user request
  or when troubleshooting.

---

### 3.4 `diagnostics.captureSnapshot`

Request a debug snapshot of the engine’s state for support tools.

**Request**

```json
{
  "command": "diagnostics.captureSnapshot",
  "options": {
    "includeGraph": true,
    "includePlugins": true,
    "includeRecentEvents": true
  }
}
```

**Response**

```json
{
  "replyTo": "diagnostics.captureSnapshot",
  "snapshotId": "diagSnap:2025-11-18T12:34:56Z"
}
```

Semantics:

- The actual snapshot contents are delivered asynchronously via a separate
  event (`diagnostics.snapshotReady`) to avoid blocking the IPC request.
- Intended more for:
  - debug builds,
  - user-triggered “collect diagnostics for support” features,
  - not for continuous use in low-latency production sessions.

---

## 4. Events (Signal → Pulse)

Diagnostics is primarily event-driven. Events are emitted at low frequency
relative to the audio callback rate.

### 4.1 `diagnostics.engineStats`

Periodic engine-wide summary stats.

**Payload**

```json
{
  "event": "diagnostics.engineStats",
  "windowMs": 1000,
  "engine": {
    "cpuLoadPercent": 15.3,
    "xruns": 0,
    "bufferUnderruns": 0,
    "averageCallbackMs": 0.7,
    "maxCallbackMs": 1.1
  },
  "latency": {
    "inputMs": 5.0,
    "outputMs": 5.0,
    "engineMs": 0.5
  }
}
```

Semantics:

- Emitted at a regular interval (e.g. once per second) while the engine is
  running.
- Pulse can:
  - drive CPU meters,
  - colour code engine health,
  - trigger gentle warnings.

---

### 4.2 `diagnostics.cohortStats`

Summary stats per cohort (real-time vs anticipative groups).

**Payload**

```json
{
  "event": "diagnostics.cohortStats",
  "windowMs": 1000,
  "cohorts": [
    {
      "cohortId": "cohort:rt:master",
      "mode": "realtime",
      "cpuLoadPercent": 8.1,
      "xruns": 0
    },
    {
      "cohortId": "cohort:anticipative:mix",
      "mode": "anticipative",
      "cpuLoadPercent": 11.4,
      "xruns": 0
    }
  ]
}
```

Semantics:

- Allows Pulse/Aura to show:
  - real-time headroom vs background rendering,
  - where load is accumulating,
  - which parts of the graph are under stress.

---

### 4.3 `diagnostics.channelStats`

Optional per-channel stats (only at `detailed` or `debug` levels).

**Payload**

```json
{
  "event": "diagnostics.channelStats",
  "windowMs": 1000,
  "channels": [
    {
      "channelId": "ch:track:42",
      "cpuLoadPercent": 4.2,
      "xruns": 0
    }
  ]
}
```

Semantics:

- May be sampled less frequently than `engineStats` to reduce overhead.
- Useful for performance inspector views or “troublesome track” highlighting.

---

### 4.4 `diagnostics.nodeStats`

Optional per-node stats (only at `detailed` or `debug` levels).

**Payload**

```json
{
  "event": "diagnostics.nodeStats",
  "windowMs": 1000,
  "nodes": [
    {
      "nodeId": "node:track:42:plugin:1",
      "cpuLoadPercent": 2.8,
      "xruns": 0
    }
  ]
}
```

Semantics:

- Particularly helpful for:
  - identifying heavy plugins,
  - verifying effectiveness of anticipative scheduling.

---

### 4.5 `diagnostics.underrun`

An audio buffer underrun has occurred.

**Payload**

```json
{
  "event": "diagnostics.underrun",
  "timestamp": "2025-11-18T12:34:56.789Z",
  "context": {
    "callbackMs": 3.5,
    "bufferDurationMs": 2.7,
    "engineCpuLoadPercent": 92.0
  }
}
```

Semantics:

- An underrun is a serious event; audio may glitch/pop/drop.
- Pulse should:
  - increment visible underrun counters,
  - optionally suggest to the user:
    - increasing buffer size,
    - disabling heavy plugins,
    - reviewing anticipative vs real-time allocation.

---

### 4.6 `diagnostics.warning`

General non-fatal diagnostic warning.

**Payload**

```json
{
  "event": "diagnostics.warning",
  "code": "highEngineLoad",
  "message": "Engine CPU load has exceeded 80% over the last 5 seconds.",
  "details": {
    "cpuLoadPercent": 82.5
  }
}
```

Examples:

- sustained high CPU load,
- frequent but non-fatal xruns in anticipative cohorts,
- suspicious plugin behaviour detected heuristically.

---

### 4.7 `diagnostics.error`

Serious diagnostics-level error that is not yet a fatal engine crash.

**Payload**

```json
{
  "event": "diagnostics.error",
  "code": "schedulingOverrun",
  "message": "Scheduling overrun detected in real-time cohort.",
  "details": {
    "cohortId": "cohort:rt:master"
  }
}
```

Semantics:

- Pulse should treat this as a high-priority signal:
  - present warnings,
  - log the condition,
  - potentially recommend safe actions or open a performance inspector view.

---

### 4.8 `diagnostics.snapshotReady`

A previously requested diagnostics snapshot is ready.

**Payload**

```json
{
  "event": "diagnostics.snapshotReady",
  "snapshotId": "diagSnap:2025-11-18T12:34:56Z",
  "snapshot": {
    "engine": {
      "cpuLoadPercent": 15.3,
      "xruns": 2
    },
    "hardware": {
      "sampleRate": 48000,
      "bufferSize": 256
    },
    "graphSummary": {
      "channelCount": 32,
      "nodeCount": 160
    },
    "recentEvents": [
      {
        "timestamp": "2025-11-18T12:34:50Z",
        "kind": "underrun"
      }
    ]
  }
}
```

Semantics:

- Intended primarily for:
  - debug logs,
  - exporting diagnostics with crash reports.

The exact snapshot structure may evolve; Pulse should treat `snapshot` as an
opaque but inspectable blob and not rely on every field being present.

---

## 5. Error Handling

Diagnostics commands may fail for reasons such as:

- unsupported detail level,
- engine not running when a summary is requested,
- temporary inability to gather stats (e.g. due to resource constraints).

Example error:

```json
{
  "error": {
    "code": "unsupportedLevel",
    "message": "Diagnostics level 'debug' is not available in this build.",
    "domain": "diagnostics",
    "details": {
      "requestedLevel": "debug"
    }
  }
}
```

Pulse should degrade gracefully, e.g.:

- falling back to `basic` level,
- hiding advanced performance inspector features.

---

## 6. Relationship to Other Signal Domains

- **signal.engine**  
  Engine state directly affects diagnostics; some errors may be mirrored between
  `engine` and `diagnostics` events.

- **signal.graph**  
  Node and Channel stats reference IDs defined in the Graph domain. Pulse uses
  this to show performance per Track/Bus/Node.

- **signal.hardware**  
  Latency and underrun behaviour are directly related to hardware config
  (sample rate, buffer size, device quality). Diagnostics helps users understand
  the trade-offs.

- **signal.plugin**  
  Many performance issues are plugin-driven; Diagnostics events can reference
  `pluginInstanceId` or Node IDs associated with plugins to highlight heavy or
  unstable plugins.

---

## 7. Extensibility

Future additions might include:

- more granular breakdowns (per voice, per Lane, per Clip),
- correlation of diagnostics with Composer telemetry (e.g. “this plugin is
  frequently heavy for many users”),
- persistent diagnostics ring-buffer for post-mortem analysis,
- user-accessible profiling sessions that integrate with external tools.

These should remain consistent with the principle that Diagnostics is:

- sampled,
- light-weight,
- and descriptive, not prescriptive.
