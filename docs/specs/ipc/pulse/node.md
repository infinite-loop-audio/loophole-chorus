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
[Mixer & Channel Architecture](../../architecture/b06-mixer-and-channel-architecture.md).

Aura manipulates Nodes at the level of the user's timeline/mixer editing intent;
Pulse owns the authoritative DSP graph and enforces constraints.

**Note**: Node creation for plugin nodes is often initiated via the **Plugin
Library domain** (`pluginLibrary.insertPlugin`), which then calls Node domain
commands to modify the graph. The Plugin Library domain handles high-level
browser-driven insertion workflows; this domain provides the low-level Node
graph manipulation primitives.

---

## 2. Node Roles and Types

Pulse recognises a core set of Node types:

- **LaneStreamNode** (often shortened to 'LaneStream')  
  Converts the output of one or more Clip Lanes into an audio/MIDI stream.

- **PluginNode**  
  Wraps a plugin instance (VST, CLAP, AU) for instrument or effect processing.

- **StackNode**  
  A container node that holds multiple plugin variants while exposing a single logical processor in the graph. Only one variant is active at a time. StackNodes enable A/B testing, preserve automation across variants, and provide lossless fallback for missing plugins. See [StackNode Architecture](../../architecture/b05-stack-nodes.md) for full details.

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
Add a Node to a Channel's DSP graph.

Payload fields include:

- `channelId`,
- `nodeType`,
- `index` (insertion point),
- `context?: "start"|"end"|"replace"|"after"|"before"` (optional positioning hint),
- Node-type-specific initial parameters or settings.

The `context` parameter provides positioning hints for "smart insert" workflows
initiated by the Plugin Library domain (e.g., `pluginLibrary.insertPlugin`).
When provided, Pulse uses the context to determine the exact insertion point
relative to existing nodes, potentially overriding or refining the `index` value.

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

Pulse updates the graph ordering and emits `moved` (event — domain: node).

---

### 3.3 Activation and State

**`node.setEnabled`**  
Enable or disable a Node.

Disabling removes the Node from the active DSP graph but retains its state.

**`node.setBypass`**  
Enable or disable bypass mode for capable Node types.

### 3.3.1 StackNode Variant Management

The following commands apply specifically to StackNodes:

**`node.setActiveVariant`**  
Set the active variant for a StackNode.

Payload fields:
- `nodeId` (must be a StackNode),
- `activeIndex` (zero-based index of the variant to activate).

Pulse updates the active variant and emits `activeVariantChanged` (event — domain: node). From the perspective of routing and channel topology, the StackNode continues to be treated as a single processor.

**`node.addVariant`**  
Add a new plugin variant to a StackNode.

Payload fields:
- `nodeId` (must be a StackNode),
- `pluginIdentifier` (optional; if omitted, creates an empty slot),
- `pluginFormat` (VST3/CLAP/AU, required if `pluginIdentifier` is provided),
- `insertIndex` (optional; defaults to end of variants list).

Pulse adds the variant and emits `variantAdded` (event — domain: node).

**`node.removeVariant`**  
Remove a variant from a StackNode.

Payload fields:
- `nodeId` (must be a StackNode),
- `variantIndex` (zero-based index of the variant to remove).

Pulse removes the variant. If the removed variant was active, Pulse activates variant 0 (or the next available variant). Pulse emits `variantRemoved` (event — domain: node).

**`node.markVariantMissing`**  
Mark a variant as missing/unavailable (typically used during project load when a plugin cannot be instantiated).

Payload fields:
- `nodeId` (must be a StackNode),
- `variantIndex`,
- `missing: bool`.

Pulse marks the variant and emits `variantMetadataChanged` (event — domain: node). Missing variants preserve their original plugin identifier and state but are not instantiated in Signal.

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

**`added`** (event — domain: node)  
Node created. Includes:

- `nodeId`,
- `channelId`,
- `nodeType`,
- `index`,
- initial metadata.

**`removed`** (event — domain: node)  
Node removed from the Channel.

**`moved`** (event — domain: node)  
Node reordered within its Channel.

---

### 4.2 State and Activation Events

**`enabledChanged`** (event — domain: node)  
Node enabled/disabled.

**`bypassChanged`** (event — domain: node)  
Bypass state updated.

**`capabilitiesChanged`** (event — domain: node)  
Capabilities modified.

---

### 4.3 Fault and Metadata Events

**`faulted`** (event — domain: node)  
Indicates a plugin or DSP failure. Aura may visualise this in the UI.  
Pulse may re-route or disable the Node as required.

**`metadataChanged`** (event — domain: node)  
For presentation updates (e.g. name, icon).

### 4.4 StackNode Variant Events

The following events are emitted for StackNode variant operations:

**`activeVariantChanged`** (event — domain: node)  
The active variant of a StackNode has changed.

Payload includes:
- `nodeId`,
- `activeIndex` (new active variant index).

**`variantAdded`** (event — domain: node)  
A new variant has been added to a StackNode.

Payload includes:
- `nodeId`,
- `variantIndex`,
- `variantId`,
- `pluginIdentifier` (if provided),
- `pluginFormat` (if provided),
- `missing: bool`.

**`variantRemoved`** (event — domain: node)  
A variant has been removed from a StackNode.

Payload includes:
- `nodeId`,
- `variantIndex` (the removed variant's index).

**`variantMetadataChanged`** (event — domain: node)  
Variant metadata has been updated (e.g. missing status, plugin identifier).

Payload includes:
- `nodeId`,
- `variantIndex`,
- updated metadata fields.

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

For StackNodes, snapshot entries additionally include:
- `variants: []` array, each with:
  - `variantId`,
  - `pluginIdentifier`,
  - `pluginFormat`,
  - `stateBlob` (serialised plugin state),
  - `missing: bool`,
  - optional metadata,
- `activeIndex` (zero-based index of the active variant).

### Snapshot Application

When Aura receives a `snapshot` (kind: snapshot — domain: project) that includes this domain, it MUST treat
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

Cohort semantics are described in the [Processing Cohorts](../architecture/a06-processing-cohorts-and-anticipative-rendering.md) architecture document.  
Pulse makes all cohort decisions; Aura simply reflects state visually where needed.
