# Signal IPC – Graph Domain

This document defines the **Graph** IPC domain between Pulse and Signal.

The Graph domain is responsible for:

- synchronising the **audio processing graph** from Pulse to Signal:
  - Channels (audio paths),
  - Nodes (processing elements),
  - connections and routing (including sends/returns),
  - cohort membership (anticipative vs real-time),
- applying structural changes (add/remove/reorder nodes, channel updates),
- reporting graph-level status and errors back to Pulse.

Pulse owns the **authoritative logical graph model** (`tracks`, `channels`,
`nodes`, `routing`, etc.).  
Signal owns the **engine-level graph realisation**, optimised for DSP.

Aura never talks directly to Signal; all graph changes flow **Pulse → Signal**.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Signal Graph domain:

- applies **graph snapshots** (full graph replacement),
- applies **graph deltas** (incremental changes),
- instantiates/destroys engine-side Channel and Node instances,
- configures connections between Channel inputs/outputs, Node inputs/outputs,
- assigns Channels/Nodes to processing cohorts (anticipative/real-time),
- reports graph validity and key lifecycle events back to Pulse.

### 1.2 Non-Goals

The Graph domain does **not**:

- define higher-level project concepts (Tracks, Lanes, Clips),
- manage parameter values or automation (those go through dedicated parameter/
  automation IPC domains and gesture channels),
- manage media decoding or file I/O directly (`signal.media`),
- handle plugin lifecycle details (`signal.plugin`),
- choose which Channels should exist — Pulse decides; Signal just realises.

---

## 2. Identity & Mapping

Pulse and Signal must agree on stable identifiers for:

- **Channel IDs** – correspond to Pulse Channels (which may or may not be
  directly visible as Tracks in Aura).
- **Node IDs** – correspond to Pulse Nodes (FaderNode, SendNode, InstrumentNode,
  SamplerNode, PluginNode, etc.).
- **Cohort IDs** – represent logical processing groups used for anticipative and
  real-time scheduling.

Signal does not invent IDs; it **accepts IDs from Pulse** and treats them as
opaque keys, while maintaining its own internal handles.

Example:

```json
{
  "channelId": "ch:main:mix",
  "nodeId": "node:track:42:fader",
  "cohortId": "cohort:rt:master"
}
```

The exact ID formats are defined in Pulse’s model/docs; the Graph domain only
requires that they are stable and unique for the duration of a session.

---

## 3. Message Flow Overview

Typical graph synchronisation flow:

1. Pulse builds/upgrades its internal graph model for the current project.
2. Pulse sends `graph.applySnapshot` to Signal with the full graph structure.
3. Signal:
   - validates the snapshot,
   - instantiates Channels and Nodes,
   - wires connections,
   - assigns cohorts,
   - responds with success or an error summary.
4. During editing, Pulse sends `graph.applyDelta` to reflect incremental changes
   (add/remove/reorder/move).
5. Signal:
   - applies changes atomically where possible,
   - may emit `graph.changed` or error events if something fails.

Graph changes must be considered **eventual**: Pulse should not assume a change
is live until Signal confirms success, or reports an error.

---

## 4. Graph Model (Engine Perspective)

### 4.1 Channels

In Signal, a **Channel** is:

- a processing path with:
  - a sequence of Nodes,
  - an input and output bus,
  - an optional name and metadata,
- not necessarily tied 1:1 to any Track:
  - Track Channels,
  - Bus/Group Channels,
  - Master Channels,
  - FX Returns,
  - hardware IO Channels.

Pulse provides all Channel definitions; Signal creates engine instances for each.

### 4.2 Nodes

A **Node** is a processing element in a Channel, e.g.:

- FaderNode (gain),
- PanNode,
- SendNode / ReturnNode,
- InstrumentNode / SamplerNode,
- PluginNode,
- StackNode (container for multiple plugin variants; appears as a single processor in the graph),
- MeterNode / AnalyserNode.

The type system and semantics are defined in Pulse’s Node architecture and IPC
docs; the Graph domain sees Nodes as:

```json
{
  "nodeId": "node:track:42:plugin:1",
  "channelId": "ch:track:42",
  "kind": "PluginNode",
  "config": {
    "pluginInstanceId": "plugin:xyz",
    "bypass": false
  },
  "position": 2
}
```

Signal is responsible for mapping `kind` → actual engine implementation.

#### 4.2.1 StackNode Variant Lifecycle

StackNodes contain multiple plugin variants, but only one variant is active at a time:

- **Active variant**: receives audio and MIDI, processes in the graph like any other processor node.
- **Inactive variants**: may be loaded in memory (during user interaction) but do not receive audio/MIDI.
- **Variant lifecycle**: Signal typically instantiates only the active variant. When Pulse instructs Signal to enter "Engaged" mode (e.g. during A/B switching), Signal may load all variants concurrently. Variants are unloaded when no longer required (Cooling/Idle states).
- **Switching**: variant activation must be glitch-minimised, ideally performed on buffer boundaries to maintain deterministic processing and correct latency reporting.
- **Missing variants**: variants with `missing: true` are not instantiated; Signal creates placeholders only.

From the graph perspective, a StackNode appears as a **single node** in the processing chain, regardless of how many variants it contains or which variant is active.

### 4.3 Cohorts

A **Cohort** groups Channels/Nodes for **anticipative** or **real-time**
processing (as defined in the Cohorts architecture doc):

- Real-time cohorts:
  - input monitoring,
  - live instruments,
  - UI-open plugin Nodes,
  - other non-deterministic or latency-sensitive paths.
- Anticipative cohorts:
  - deterministic, pre-renderable portions of the graph.

Pulse assigns Channels/Nodes to cohorts by sending cohort metadata; Signal
schedules them accordingly.

---

## 5. Commands (Pulse → Signal)

### 5.1 `graph.applySnapshot`

Apply a full graph snapshot: Channels, Nodes, connections, and cohort mapping.

**Request**

```json
{
  "command": "graph.applySnapshot",
  "graphId": "graph:project:current",
  "channels": [
    {
      "channelId": "ch:master",
      "name": "Master",
      "inputs": {
        "sources": ["ch:bus:drums", "ch:bus:music"]
      },
      "outputs": {
        "hardwareOut": "hw:mainOut"
      },
      "nodes": [
        {
          "nodeId": "node:master:meterIn",
          "kind": "MeterNode",
          "position": 0,
          "config": {}
        },
        {
          "nodeId": "node:master:fader",
          "kind": "FaderNode",
          "position": 1,
          "config": {
            "initialGainDb": 0.0
          }
        },
        {
          "nodeId": "node:master:meterOut",
          "kind": "MeterNode",
          "position": 2,
          "config": {}
        }
      ]
    }
  ],
  "cohorts": [
    {
      "cohortId": "cohort:rt:master",
      "mode": "realtime",
      "channelIds": ["ch:master"]
    }
  ]
}
```

**Semantics**

- Signal should treat this as a **complete replacement** of any existing graph.
- On success, previous Channels/Nodes become invalid (unless reused internally
  in an optimisation).
- This command is ideal on project load, large topology changes, or recovery.

**Response**

- Success:
  ```json
  { "replyTo": "graph.applySnapshot", "ok": true }
  ```
- Error:
  - Use standard error envelope with details of failed Channels/Nodes.

---

### 5.2 `graph.applyDelta`

Apply incremental changes to the graph.

**Request**

```json
{
  "command": "graph.applyDelta",
  "graphId": "graph:project:current",
  "changes": [
    {
      "op": "addChannel",
      "channel": {
        "channelId": "ch:fx:delay",
        "name": "FX – Delay",
        "inputs": {
          "sources": []
        },
        "outputs": {
          "toChannels": ["ch:master"]
        },
        "nodes": []
      }
    },
    {
      "op": "addNode",
      "node": {
        "nodeId": "node:fx:delay:plugin",
        "channelId": "ch:fx:delay",
        "kind": "PluginNode",
        "position": 0,
        "config": {
          "pluginInstanceId": "plugin:someDelay"
        }
      }
    },
    {
      "op": "connectChannel",
      "fromChannelId": "ch:track:snare",
      "toChannelId": "ch:fx:delay"
    }
  ]
}
```

Supported operations (non-exhaustive, but core set):

- `addChannel`
- `updateChannel` (name, metadata, IO)
- `removeChannel`
- `addNode`
- `updateNode`
- `removeNode`
- `reorderNodes` (within a Channel)
- `connectChannel` (route output of one Channel to another’s input)
- `disconnectChannel`
- `setCohorts` (reassign Channels/Nodes to cohorts)

**Semantics**

- Signal should apply the **entire delta as an atomic operation** if possible.
- On error:
  - either reject the whole delta (preferred for simplicity), or
  - apply partial ops and report which ones failed (implementation detail, but
    must be documented clearly).
- Pulse should not assume the graph changed until it receives successful
  acknowledgement.

---

### 5.3 `graph.setCohorts`

Set cohort assignments separately from topology changes.

This may be used when only processing allocation changes are needed.

**Request**

```json
{
  "command": "graph.setCohorts",
  "cohorts": [
    {
      "cohortId": "cohort:rt:inputs",
      "mode": "realtime",
      "channelIds": ["ch:track:vocal", "ch:track:guitar"],
      "nodeIds": []
    },
    {
      "cohortId": "cohort:anticipative:mix",
      "mode": "anticipative",
      "channelIds": ["ch:bus:drums", "ch:bus:music"],
      "nodeIds": []
    }
  ]
}
```

**Semantics**

- Replaces existing cohort assignments with the provided set.
- Signal is free to further optimise or subdivide internally, but must preserve
  the intent:
  - real-time designated Channels/Nodes must remain low-latency and not be
    blocked by anticipative work.

---

### 5.4 `graph.querySummary`

Request a summary of the current engine graph.

**Request**

```json
{
  "command": "graph.querySummary"
}
```

**Response**

```json
{
  "replyTo": "graph.querySummary",
  "graphId": "graph:project:current",
  "channels": [
    {
      "channelId": "ch:master",
      "nodeCount": 3,
      "inputChannelIds": ["ch:bus:drums", "ch:bus:music"],
      "outputChannelIds": [],
      "hardwareOutputs": ["hw:mainOut"]
    }
  ],
  "cohorts": [
    {
      "cohortId": "cohort:rt:master",
      "mode": "realtime",
      "channelIds": ["ch:master"]
    }
  ]
}
```

Semantics:

- Intended for debugging, diagnostics, and reconciliation in Pulse.
- Not required for normal editing flows, but useful for recovery and tooling.

---

## 6. Events (Signal → Pulse)

### 6.1 `graph.applied`

Emitted when a snapshot or delta has been successfully applied.

**Payload**

```json
{
  "event": "graph.applied",
  "graphId": "graph:project:current",
  "changeId": "delta:2025-11-18T12:34:56Z"
}
```

Pulse may include its own `changeId` in `graph.applyDelta`; Signal should echo
it back here if provided.

---

### 6.2 `graph.invalid`

Emitted when Signal detects a structural issue in the graph that prevents
correct operation.

**Payload**

```json
{
  "event": "graph.invalid",
  "graphId": "graph:project:current",
  "issues": [
    {
      "code": "cycleDetected",
      "message": "Feedback loop detected between channels.",
      "details": {
        "channelIds": ["ch:bus:delay", "ch:bus:reverb"]
      }
    }
  ]
}
```

Semantics:

- Does **not** necessarily stop audio; implementation-specific.
- Pulse should interpret this as a serious configuration problem and may:
  - adjust the model to break cycles,
  - notify the user,
  - avoid further risky changes until resolved.

---

### 6.3 `graph.warning`

Non-fatal graph-level issues.

**Payload**

```json
{
  "event": "graph.warning",
  "code": "unconnectedChannel",
  "message": "Channel has no outputs and will be silent.",
  "details": {
    "channelId": "ch:fx:unused"
  }
}
```

Examples:

- channels without outputs,
- nodes with unsupported configurations,
- cohort oversubscription hints.

---

### 6.4 `graph.cohortsChanged`

Cohort assignments have changed (internally or as a result of `graph.setCohorts`).

**Payload**

```json
{
  "event": "graph.cohortsChanged",
  "cohorts": [
    {
      "cohortId": "cohort:rt:inputs",
      "mode": "realtime",
      "channelIds": ["ch:track:vocal", "ch:track:guitar"],
      "nodeIds": []
    }
  ]
}
```

Signal may emit this event if it internally adjusts cohort membership in
response to performance constraints, though such behaviour should be rare and
well-defined.

---

## 7. Error Handling

Graph-related commands follow the standard IPC error format.

Typical error scenarios:

- invalid Channel or Node IDs,
- attempts to remove Channels/Nodes that are in use or referenced,
- creating cycles that violate engine constraints,
- cohort assignments that conflict with hardware or engine limitations.

Example error:

```json
{
  "error": {
    "code": "graphCycle",
    "message": "Adding connection would create a feedback loop.",
    "domain": "graph",
    "details": {
      "fromChannelId": "ch:fx:delay",
      "toChannelId": "ch:bus:drums"
    }
  }
}
```

Pulse should treat these as authoritative and adjust its model accordingly.

---

## 8. Relationship to Other Signal Domains

- **signal.engine**  
  Graph operations are typically only meaningful when the engine has a valid
  configuration. However, some implementations may allow pre-graph building
  before starting the engine.

- **signal.transport**  
  Transport playback uses the realised graph. Graph changes may temporarily
  affect audible output (e.g. when moving Nodes) and should ideally be
  glitch-minimised in the engine.

- **signal.media**  
  Some Nodes (e.g. SamplerNode, InstrumentNode) require media resources. Media
  availability is handled in the Media domain; graph application should
  gracefully handle missing media by creating Nodes in a ‘pending’ state.

- **signal.plugin**  
  Plugin-backed Nodes (PluginNode, InstrumentNode) rely on plugin instances
  managed by the Plugin domain. Graph operations must respect plugin lifecycle
  constraints.

- **signal.diagnostics**  
  Per-node and per-channel performance metrics are reported via Diagnostics,
  often keyed by the same Node/Channel IDs used in Graph.

---

## 9. Extensibility

Future extensions may include:

- explicit parallelism hints for Channels/Nodes,
- alternate graph partitions for multi-engine configurations,
- partial graph snapshots per Section or Scene,
- per-track custom processing graphs (e.g. clip-based FX containers).

Such features must be additive to the existing model and avoid breaking
backwards compatibility with the Graph domain protocol.

The Graph domain should remain a **structural** description of the engine’s
processing topology, with detailed Node behaviour and parameters handled by
their respective dedicated domains.
