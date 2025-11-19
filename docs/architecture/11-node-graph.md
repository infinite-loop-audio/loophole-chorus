# Node Graph Architecture

This document defines the conceptual model for **nodes** in Loophole. It
covers what a node is, how nodes are identified and owned, how they
form Channel chains, how they relate to Tracks, Clips and Lanes, and how they
interact with parameters, automation, Pulse, Signal and Composer.

This is an architectural document. It describes shapes, responsibilities and
relationships, not JUCE implementation details or IPC formats.

---

## Contents

- [1. Overview](#1-overview)
- [2. Node Responsibilities](#2-node-responsibilities)
- [3. Node Types](#3-node-types)
  - [3.1 Instruments](#31-instruments)
  - [3.2 Audio Effects](#32-audio-effects)
  - [3.3 LaneStreamNode](#33-lanestreamnode)
  - [3.4 Utility and Analysis Nodes](#34-utility-and-analysis-nodes)
  - [3.5 Device/IO Nodes](#35-deviceio-nodes)
- [4. Node Identity](#4-node-identity)
  - [4.1 Node IDs](#41-node-ids)
  - [4.2 Plugin Identity vs Node Identity](#42-plugin-identity-vs-node-identity)
  - [4.3 Stability Across Edits](#43-stability-across-edits)
- [5. Node Graphs and Channels](#5-node-graphs-and-channels)
  - [5.1 Serial Chains](#51-serial-chains)
  - [5.2 Relationship to Tracks and Clips](#52-relationship-to-tracks-and-clips)
  - [5.3 LaneStreamNode as First-Class Nodes](#53-lanestreamnode-as-first-class-nodes)
- [6. Parameters and Nodes](#6-parameters-and-nodes)
  - [6.1 Parameter Ownership](#61-parameter-ownership)
  - [6.2 Parameter Discovery](#62-parameter-discovery)
  - [6.3 Automation Targets](#63-automation-targets)
- [7. Node Lifecycle](#7-node-lifecycle)
  - [7.1 Creation](#71-creation)
  - [7.2 Replacement](#72-replacement)
  - [7.3 Removal](#73-removal)
  - [7.4 Bypassing](#74-bypassing)
- [8. Interaction with Pulse](#8-interaction-with-pulse)
- [9. Interaction with Signal](#9-interaction-with-signal)
- [10. Interaction with Composer](#10-interaction-with-composer)
- [11. Interaction with Aura](#11-interaction-with-aura)
- [12. Future Extensions](#12-future-extensions)

---

## 1. Overview

Nodes are the core building blocks of audio and control processing in
Loophole. They live inside **Channels** and are executed by **Signal** in a
strictly serial chain. Nodes include:

- instruments,
- audio effects,
- LaneStreamNode(s) (often shortened to 'LaneStream'),
- utility and analysis nodes,
- device/IO abstractions.

Nodes:

- own parameters,
- receive audio, MIDI or other control streams,
- produce audio and/or control streams,
- participate in automation and parameter mapping.

Pulse manages node graphs as part of the project model. Signal instantiates
and runs nodes. Aura visualises and edits them. Composer enriches
node-related metadata but never controls node behaviour directly.

---

## 2. Node Responsibilities

A node:

- consumes zero or more input streams (audio, MIDI, control),
- produces zero or more output streams (audio, MIDI, control),
- exposes a set of parameters,
- responds to parameter changes and automation,
- performs a deterministic transformation on its inputs.

Nodes must not:

- own project-level state,
- store Clip or Track structures,
- perform non-deterministic transformations dependent on UI state,
- manage their own routing beyond their defined inputs/outputs.

---

## 3. Node Types

### 3.1 Instruments

Instrument nodes:

- receive MIDI (and optionally other control streams),
- produce audio,
- may expose large parameter sets,
- may support MPE, per-note expressions and other advanced control schemes.

An instrument is just a node from the engine's point of view. The key
distinction is the presence of MIDI input and audio output.

### 3.2 Audio Effects

Audio effect nodes:

- receive audio (and optionally MIDI/control),
- produce audio,
- may be pre- or post-fader (depending on chain position),
- include EQs, dynamics, saturators, reverbs, delays and so on.

Effects may also expose sidechain inputs; those are represented as additional
inputs in the node model.

### 3.3 StackNode

**StackNode** — a container node that holds multiple plugin variants (e.g. different EQs or compressors) while exposing a single logical processor in the graph. Only one variant is active at a time. StackNodes enable A/B testing of plugins, support lossless fallback behaviour for missing plugins, and preserve all plugin automation/state across variants. See [StackNode Architecture](./12-stack-nodes.md) for full details.

### ARA-Capable Plugin Nodes

A PluginNode may declare `supportsAra = true`, indicating it can host
ARA-style clip-oriented editing. Such nodes receive ARA region metadata
from Pulse, including:

- regionId

- clipId, laneId

- bounds in clip (musical time)

- optional groupId for multi-lane editing

Signal does not manipulate musical-time data; it receives resolved
sample-accurate region descriptors from Pulse. The backend may realise
ARA groups as either:

- a single node with multiple audio inputs, or

- multiple nodes internally coordinated by the plugin.

This behaviour is implementation-specific and not part of the Pulse
contract.

### 3.4 LaneStreamNode

LaneStreamNode(s) (often shortened to 'LaneStream') are special-purpose nodes that:

- receive audio from Clip audio Lanes on a Track,
- appear at (or near) the top of the Channel node graph by default,
- provide per-Lane mixing controls (gain, pan, phase, mute, etc.),
- may be split so different sets of Lanes feed different LaneStreamNodes.

LaneStreamNodes are first-class nodes:

- they have parameters,
- they can be automated,
- they can be reordered (subject to constraints defined by Pulse),
- they can be inspected in Aura like any other node.

### 3.5 Utility and Analysis Nodes

Utility nodes include:

- gain and trim nodes,
- pan and balance nodes,
- meter nodes,
- channel format converters (mono/stereo/etc.).

Analysis nodes include:

- spectrum analysers,
- loudness meters,
- correlation meters.

These nodes may be native to Signal or plugin-based. In both cases they
obey the same node contract.

### 3.6 Device/IO Nodes

Device/IO nodes represent:

- audio interface inputs and outputs,
- hardware insert send/return points,
- external device control endpoints.

They are the bridge between the abstract engine graph and the actual system
hardware and device layer.

---

## 4. Node Identity

### 4.1 Node IDs

Each node has a stable ID within its Channel:

- assigned by Pulse on creation,
- unique per Channel,
- stable across project saves, undo/redo, and minor edits.

Node IDs are used in:

- parameter IDs (see [@chorus:/docs/architecture/10-parameters.md](10-parameters.md)),
- automation targets,
- routing specifications,
- graph update messages to Signal.

### 4.2 Plugin Identity vs Node Identity

A node may be:

- a plugin instance (VST3, CLAP, AU), or
- a built-in DSP unit (utility, LaneStreamNode, device IO).

**Plugin identity** refers to:

- vendor/product,
- format,
- version.

**Node identity** refers to:

- this specific instance in this Channel in this project.

Multiple nodes can share the same plugin identity (several instances of the
same plugin in different places), but they always have distinct node IDs.

### 4.3 Stability Across Edits

Node IDs MUST remain stable under:

- Track duplication,
- Clip editing,
- simple reordering of nodes,
- parameter edits.

Node IDs MAY change under:

- Channel reallocation across engines (future),
- certain destructive edits that fundamentally replace nodes.

Pulse is responsible for ensuring that automation and parameter bindings are
updated if node IDs are changed.

---

## 5. Node Graphs and Channels

### 5.1 Serial Chains

Each Channel has a serial node graph:

```
[ LaneStreamNode(s) ] → [ Instruments ] → [ Effects ] → [ Utility/Metering ] → [ Output ]
```

There is no concurrent parallel graph at this level; parallelism is achieved via:

- LaneStreamNodes feeding different nodes,
- multiple Channels,
- sends/returns (future),
- routing and summing between Channels.

Pulse defines the graph; Signal executes it.

### 5.2 Relationship to Tracks and Clips

Nodes:

- live in Channels,
- do not live in Tracks or Clips.

Tracks:

- reference Channels,
- cause Channels to exist (by requiring audio/instruments),
- never embed nodes directly.

Clips:

- define Lanes and content,
- feed LaneStreamNodes and nodes via Channel routing,
- never own nodes.

### 5.3 LaneStreamNode as First-Class Nodes

LaneStreamNodes are treated as nodes in the graph:

- they may appear at different positions in the graph,
- they can be moved before or after specific effect nodes (subject to rules),
- they expose their own parameters for:

  - per-Lane gain/pan/phase,
  - summing modes,
  - possibly per-Lane timing/phase alignment (future).

This allows complex workflows such as:

- processing only certain Lane outputs before they are mixed with others,
- inserting effects that see only a subset of Lanes,
- creating internal "submixes" within a Track.

---

## 6. Parameters and Nodes

### 6.1 Parameter Ownership

Nodes own parameters. For each node:

- all of its parameters are attached to that node ID,
- parameter IDs are node-local and combined with the node ID to form
  fully qualified parameter IDs,
- Pulse is responsible for reflecting these parameters in the global parameter
  model.

LaneStreamNode parameters are also node-owned:

- per-Lane level, pan, mute, phase, etc.,
- aggregated or per-Lane views for automation.

### 6.2 Parameter Discovery

When a node is created or replaced:

- Signal queries the plugin/DSP node for its parameters,
- Pulse receives the parameter list,
- Pulse merges this with existing alias tables and Composer metadata,
- Pulse exposes the resolved parameter set to Aura.

Parameter discovery must be deterministic for a given plugin/DSP implementation.

### 6.3 Automation Targets

Automation Lanes target node parameters through:

- Channel ID,
- node ID,
- parameter ID.

Automation is scoped at the Clip level but resolves to node parameters at
playback. Node replacement and plugin updates may change the available
parameters; Pulse must remap automation where possible.

---

## 7. Node Lifecycle

### 7.1 Creation

Nodes can be created by:

- user inserting a plugin or built-in node in Aura,
- implicit creation (LaneStreamNode when first audio lane appears),
- Track templates and presets,
- project loading.

Pulse adds a node to the Channel graph, assigns an ID, and instructs Signal
to instantiate it.

### 7.2 Replacement

Node replacement occurs when:

- a user swaps one plugin for another,
- a plugin format is changed (VST3 → CLAP),
- a plugin is updated and old binary is no longer available,
- a node is upgraded to a different implementation.

Pulse:

- creates a new node instance,
- reassigns automation via alias and Composer metadata where possible,
- updates the Channel graph accordingly.

Signal is instructed to tear down the old node and load the new one.

When a plugin is missing during project load, Pulse wraps it in a **StackNode** to preserve the original plugin state and allow seamless fallback. See [StackNode Architecture](./12-stack-nodes.md) for details on how StackNodes handle missing plugins and enable non-destructive plugin replacement.

### 7.3 Removal

Removing a node:

- removes it from the Channel graph,
- invalidates its parameters,
- may orphan automation lanes (Pulse should warn and offer reassignment),
- may necessitate rebalancing levels or adjusting Lane routing.

Removed nodes are not kept in memory; state persists only via project
serialisation or explicit user actions (e.g. presets).

### 7.4 Bypassing

Nodes may be bypassed:

- bypass is a parameter state, not structural removal,
- bypass should be automatable (subject to performance constraints),
- bypass semantics should be consistent across plugin formats where possible.

Bypass is applied in Signal but controlled and persisted by Pulse.

---

## 8. Interaction with Pulse

Pulse:

- owns the node graph model for each Channel,
- creates, reorders, replaces and removes nodes,
- manages node IDs and mapping,
- owns parameter state and automation,
- determines how LaneStreamNodes are created and routed,
- uses Composer to interpret plugin and parameter semantics.

Nodes never directly "talk" to Pulse; all communication is mediated through
Signal and the node/parameter model.

---

## 9. Interaction with Signal

Signal:

- instantiates node implementations (plugins/DSP),
- manages plugin lifecycles (initialise, prepare, process, release),
- applies parameter updates at audio rate,
- runs node graphs in real time,
- reports failures or status changes back to Pulse.

Signal does not interpret:

- semantic roles,
- aliasing,
- routing rules beyond execution,
- plugin categorisation.

All those concerns are handled by Pulse (and Composer).

---

## 10. Interaction with Composer

Nodes themselves are not aware of Composer. Composer metadata affects how
Pulse reasons about nodes:

- plugin identity normalisation (family, variants, formats),
- parameter metadata (names, ranges, units, scaling),
- semantic roles (e.g. `filter.cutoff`, `fx.mix`),
- mapping suggestions when replacing or updating nodes.

Composer helps Pulse preserve node behaviour at the project level when the
underlying plugin or DSP implementation changes.

---

## 11. Interaction with Aura

Aura:

- displays node graphs in the mixer and Track inspector,
- allows inserting, reordering, bypassing and removing nodes,
- opens plugin UIs (via Signal window handles),
- exposes LaneStreamNode controls,
- surfaces Composer-enriched metadata (parameter roles, groups, categories).

Aura never directly controls node execution or routing. It sends editing
intents to Pulse and renders the resulting model.

---

## 12. Future Extensions

The node model anticipates future capabilities such as:

- multi-channel and spatial nodes,
- graph-based internal routing within a node "slot" (e.g. container
  nodes),
- macro nodes that wrap multiple internal nodes,
- dynamic node graphs per Clip or Section (with strict RT constraints),
- per-Lane insert chains (inside LaneStreamNodes),
- distributed or remote nodes.

All such features must preserve the core invariants:

- Pulse owns node structure and identity,
- Signal executes a resolved serial graph,
- Aura reflects, not invents, node configuration,
- Composer remains a metadata-only service.

