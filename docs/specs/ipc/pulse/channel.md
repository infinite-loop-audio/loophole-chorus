# Pulse Channel Domain Specification

This document defines the Channel domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse relating to Channel
configuration and the events emitted by Pulse to notify Aura of Channel-level
changes.

A **Channel** is a DSP processing container. It owns:

- an ordered sequence of Nodes that form its DSP graph,
- input and output routing,
- built-in gain/pan state,
- metering configuration,
- default LaneStreamNode(s) (often shortened to 'LaneStream') (created when Lanes exist),
- analysis nodes (if required).

Channels are often created for and associated with Tracks (track-owned Channels),
but Channels may also be standalone and not bound to any Track, for example:
master output Channels, bus/FX Channels, send/return Channels, hardware I/O
Channels, and analysis-only or utility Channels.

Pulse is the authoritative owner of Channel identity, structure and configuration.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Channel Roles and Structure](#2-channel-roles-and-structure)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Creation and Removal](#31-creation-and-removal)  
  - [3.2 Input and Output Configuration](#32-input-and-output-configuration)  
  - [3.3 Gain, Pan and Fader State](#33-gain-pan-and-fader-state)  
  - [3.4 Metering](#34-metering)  
  - [3.5 Channel Graph Reset](#35-channel-graph-reset)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Structural Events](#41-structural-events)  
  - [4.2 Configuration Events](#42-configuration-events)  
  - [4.3 Metering Events](#43-metering-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)

---

## 1. Overview

Channels define DSP processing contexts. Every Node is owned by exactly one
Channel. A Channel is responsible for:

- receiving audio/MIDI streams from Lanes (for track-owned Channels),
- applying the Node graph (plugin nodes, built-in DSP, routing nodes, etc.),
- producing output streams for the project-wide routing graph,
- emitting metering and analysis data,
- integrating with cohort assignment decisions.

The overall mixing model and console semantics are described in
[Mixer & Channel Architecture](../../architecture/B06-mixer-and-channel-architecture.md).

Track-owned Channels are created when Tracks request them (Track domain's
`track.setChannelEnabled` command).  
Other Channels (busses, sends, returns, master, I/O, etc.) are defined at the
project/routing level and exist independently of Tracks.

---

## 2. Channel Roles and Structure

Each Channel has:

- a stable `channelId`,
- an optional `trackId` for Track-owned Channels,
- an optional `role` field for special Channels such as `"track"`, `"master"`,
  `"bus"`, `"fx"`, `"send"`, `"return"`, `"monitor"`, `"io"`, etc.,
- an ordered list of `nodeId`s representing the Node graph (which may include StackNodes; StackNodes are semantically equivalent to a single processor in the chain),
- input configuration (Track input sources, audio/MIDI),
- output configuration (main output, bus routing, sidechain sends),
- fader state (gain, pan, balance),
- meter configuration (peak, RMS, LUFS, analysis parameters).

Channel roles influence routing and UI presentation, but all Channels use the
same underlying fader model: FaderNode and PanNode in the Node graph. There are
no special "VCA Channels" at IPC level; VCA-style control behaviour is
expressed via parameter/control groups in the Parameter domain.

The Channel domain governs channel-level configuration.  
Node operations are handled in the Node domain.

Sends are materialised as SendNodes within the Channel's Node graph. A SendNode taps the signal at its position in the Node list and routes a copy to a target Channel, allowing pre/post-fader behaviour and arbitrary tap points purely via graph topology.

MIDI CC levels (e.g. CC7, CC10, CC11) are handled as parameters and via
Lane/Parameter domains, not as separate mixer fader types. The Channel domain
deals with audio-level faders (FaderNodes), not MIDI CC-specific faders.

---

## 3. Commands (Aura → Pulse)

### 3.1 Creation and Removal

Track-owned Channels are created/removed via Track domain commands.  
Standalone Channels (busses, master, etc.) are created via routing/project-level
commands (defined in the Routing domain).

The Channel domain receives no direct create/delete commands.

However, Aura may issue Channel-specific operations once a Channel exists.

---

### 3.2 Input and Output Configuration

**`channel.setInput`**  
Set input source(s) for the Channel.

Payload fields include:

- `channelId`,
- physical input identifiers (for audio),
- virtual/MIDI sources,
- optional input modes.

Pulse validates that the input exists and is compatible.

**`channel.setOutput`**  
Set the output routing target for the Channel.

Fields include:

- `channelId`,
- output bus or master output target.

Pulse recomputes routing if necessary.

**`channel.setSidechainSource`**  
Define a sidechain source for Nodes that support sidechain input.

Pulse may update routing nodes accordingly.

---

### 3.3 Gain, Pan and Fader State

**`channel.setGain`**  
Set fader gain for the Channel.

**`channel.setPan`**  
Set pan/balance state.

These affect built-in fader/pan nodes or Channel-level mixing nodes.

---

### 3.4 Metering

Metering streams are created via the gesture/metering stream protocol.  
The Channel domain exposes non-stream configuration only.

**`channel.setMeteringConfig`**  
Configure meter behaviour:

- peak window,
- RMS falloff,
- LUFS integration window,
- analysis modes (if supported).

---

### 3.5 Channel Graph Reset

**`channel.resetGraph`**  
Remove all user-added Nodes and restore the Channel to its default minimal
configuration (LaneStreamNode(s) + FaderNode/PanNode).

Pulse then emits:

- `channel.graphReset`,  
- followed by Node-domain events describing the newly created Node graph.

---

## 4. Events (Pulse → Aura)

### 4.1 Structural Events

**`channel.created`**  
A Channel has been created (either due to Track enabling or via routing/project
configuration).

Includes:

- `channelId`,
- `trackId` (if track-owned),
- `role` (if applicable),
- initial node list.

**`channel.deleted`**  
The Channel has been removed (either due to Track disabling or routing/project
configuration changes).

---

### 4.2 Configuration Events

**`channel.inputChanged`**  
Input configuration updated.

**`channel.outputChanged`**  
Output routing updated.

**`channel.sidechainSourceChanged`**  
Sidechain routing updated.

**`channel.gainChanged`**  
Fader gain updated.

**`channel.panChanged`**  
Pan/balance updated.

---

### 4.3 Metering Events

Metering data itself is sent via binary meter streams, not Channel events.

Channel domain may emit:

**`channel.meteringConfigChanged`**  
Static meter configuration updated.

---

## 5. Snapshot Semantics

Channel entries in snapshots include:

- identity (`channelId`),
- associated `trackId` (if track-owned),
- `role` (if applicable),
- complete ordered list of `nodeId`s,
- input/output configuration,
- fader state,
- meter configuration.

Node snapshot data is included in the Node domain.  
Channel snapshot data binds them together.

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

## 6. Realtime and Cohort Considerations

Channel domain operations are non-realtime.  
However, Channel changes may:

- require Node graph rebuilds in Signal,
- invalidate anticipative buffers,
- trigger cohort reassignment via Pulse → Signal updates.

Cohort behaviour is governed by the [Processing Cohorts](../architecture/A06-processing-cohorts-and-anticipative-rendering.md) architecture document.

Aura never interacts with cohorts directly; Channel events may trigger UI cues
(e.g. meter resets, graph rebuild animations).
