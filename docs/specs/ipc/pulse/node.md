# Pulse Node Domain Specification

This document defines the Node domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse for adding, removing or modifying
Nodes within a Channel’s DSP graph, and the events emitted by Pulse to notify 
Aura of Node-level changes.

A **Node** is the fundamental processing unit within a Channel's DSP graph.  
Nodes may represent:

- plugin-based processors (VST, CLAP, AU wrappers),
- built-in DSP units (filters, mixers, gain, analysers),
- LaneStreamNode(s) (often shortened to 'LaneStream') (rendering output of Clip Lanes),
- routing nodes (SendNodes, ReturnNodes, sidechain joiners),
- utility nodes (latency compensators, channel mergers/splitters).

Nodes are ordered within a Channel, form part of the larger real-time/anticipative
processing cohort (PC) model, and own **parameters** defined in the Parameter domain.

Pulse is the authoritative owner of Node identity, structure, ordering and metadata.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Node Roles and Types](#2-node-roles-and-types)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Creation and Removal](#31-creation-and-removal)  
  - [3.2 Ordering and Placement](#32-ordering-and-placement)  
  - [3.3 Activation and State](#33-activation-and-state)  
  - [3.4 Parameter and Capability Updates](#34-parameter-and-capability-updates)  
  - [3.5 Identity and Presentation](#35-identity-and-presentation)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Structural Events](#41-structural-events)  
  - [4.2 State and Activation Events](#42-state-and-activation-events)  
  - [4.3 Fault and Metadata Events](#43-fault-and-metadata-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Cohort and Realtime Considerations](#6-cohort-and-realtime-considerations)

---

## 1. Overview

Nodes form the processing chain of a Channel. Each Node:

- has a stable `nodeId`,
- belongs to a specific `channelId`,
- has a Node type (PluginNode, LaneStreamNode, GainNode, AnalyzerNode, etc.),
- has input and output stream layouts,
- has parameters (defined in the Parameter domain),
- may host plugin UI state (via the UI domain),
- participates in Pulse → Signal graph construction,
- may be assigned to the live or anticipative cohort.

For an architectural overview of how Nodes participate in the mixer and
console model, see
[Mixer & Channel Architecture](../../architecture/15-mixer-and-channel-architecture.md).

Aura manipulates Nodes at the level of the user's timeline/mixer editing intent;
Pulse owns the authoritative DSP graph and enforces constraints.

---

## 2. Node Roles and Types

Pulse recognises a core set of Node types:

- **LaneStreamNode** (often shortened to 'LaneStream')  
  Converts the output of one or more Clip Lanes into an audio/MIDI stream.

- **PluginNode**  
  Wraps a plugin instance (VST, CLAP, AU) for instrument or effect processing.

- **FaderNode / PanNode**  
  Built-in Nodes that implement the Channel fader and pan. FaderNode and PanNode are explicitly realised within the Channel's Node graph, enabling pre/post-fader send behaviour to be defined purely by Node graph topology.

- **GainNode / MixerNode**  
  Built-in mixing utilities for gain adjustment and channel merging/splitting.

- **SendNode**  
  Taps the Channel signal at its position in the Node list and routes a copy to a target Channel. The position of the SendNode relative to other Nodes (particularly FaderNode) defines pre/post-fader and pre/post-other-processing behaviour.

- **ReturnNode**  
  Utility Node for FX return or specialised merge paths, if required by future designs.

- **RoutingNode**  
  Handles merges and splits (non-send routing).

- **AnalyzerNode**  
  Provides metering, spectral analysis and other diagnostic outputs.

Each Node type defines:

- capabilities,
- supported parameters,
- I/O layout,
- determinism metadata (informed by Composer),
- scheduling constraints.

---

## 3. Commands (Aura → Pulse)

### 3.1 Creation and Removal

**`node.add`**  
Add a Node to a Channel’s DSP graph.

Payload fields include:

- `channelId`,
- `nodeType`,
- `index` (insertion point),
- Node-type-specific initial parameters or settings.

Pulse allocates a `nodeId` and inserts the Node.

**`node.remove`**  
Remove a Node from a Channel.

Pulse ensures:

- parameter references are dropped,
- routing integrity is maintained,
- cohort assignments are refreshed.

---

### 3.2 Ordering and Placement

**`node.move`**  
Reorder a Node within its Channel.

Fields:

- `nodeId`,
- `newIndex`.

Pulse updates the graph ordering and emits `node.moved`.

---

### 3.3 Activation and State

**`node.setEnabled`**  
Enable or disable a Node.

Disabling removes the Node from the active DSP graph but retains its state.

**`node.setBypass`**  
Enable or disable bypass mode for capable Node types.

---

### 3.4 Parameter and Capability Updates

Parameter changes are expressed in the Parameter domain, not here.  
However, Node-level capabilities may be updated:

**`node.setCapabilities`**  
For Node types that expose optional modes (e.g. stereo/mono, oversampling, analysis resolution).

Pulse validates capability changes and applies them to the model.

---

### 3.5 Identity and Presentation

**`node.renameInstance`**  
Rename a Node for user-facing clarity (applies primarily to PluginNodes).

**`node.setIcon`**  
Optional presentation metadata for future graph/panel views.

---

## 4. Events (Pulse → Aura)

Pulse emits Node events after applying valid commands or when graph changes
propagate from higher-level model updates.

### 4.1 Structural Events

**`node.added`**  
Node created. Includes:

- `nodeId`,
- `channelId`,
- `nodeType`,
- `index`,
- initial metadata.

**`node.removed`**  
Node removed from the Channel.

**`node.moved`**  
Node reordered within its Channel.

---

### 4.2 State and Activation Events

**`node.enabledChanged`**  
Node enabled/disabled.

**`node.bypassChanged`**  
Bypass state updated.

**`node.capabilitiesChanged`**  
Capabilities modified.

---

### 4.3 Fault and Metadata Events

**`node.faulted`**  
Indicates a plugin or DSP failure. Aura may visualise this in the UI.  
Pulse may re-route or disable the Node as required.

**`node.metadataChanged`**  
For presentation updates (e.g. name, icon).

---

## 5. Snapshot Semantics

Node snapshot entries appear inside Channel snapshots or as part of a global
project snapshot and include:

- identity (`nodeId`),
- `channelId`,
- ordering index,
- Node type,
- capability metadata,
- enabled/bypass state,
- parameter schema summary (full parameter values appear in Parameter domain snapshots),
- Node-type-specific static properties.

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

## 6. Cohort and Realtime Considerations

Node domain operations are non-realtime and must not affect the audio thread.

Node changes may, however:

- require cohort reassignment,
- invalidate anticipative render buffers,
- trigger graph rebuilds in Signal.

Cohort semantics are described in the [Processing Cohorts](../architecture/10-processing-cohorts-and-anticipative-rendering.md) architecture document.  
Pulse makes all cohort decisions; Aura simply reflects state visually where needed.
