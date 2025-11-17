# Pulse Channel Domain Specification

This document defines the Channel domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse relating to Channel
configuration and the events emitted by Pulse to notify Aura of Channel-level
changes.

A **Channel** is the DSP container for a Track. It owns:

- an ordered sequence of Nodes that form its DSP graph,
- input and output routing,
- built-in gain/pan state,
- metering configuration,
- default LaneStreamNodes (created when Lanes exist),
- analysis nodes (if required).

Channels exist only when enabled for a Track.  
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

Channels define the DSP context for Tracks. Every Node is owned by exactly one
Channel. A Channel is responsible for:

- receiving audio/MIDI streams from Lanes,
- applying the Node graph (plugin nodes, built-in DSP, routing nodes, etc.),
- producing output streams for the project-wide routing graph,
- emitting metering and analysis data,
- integrating with cohort assignment decisions.

Channels are created when Tracks request them (Track domain’s
`track.setChannelEnabled` command).  
Channels do not exist independently of Tracks.

---

## 2. Channel Roles and Structure

Each Channel has:

- a stable `channelId`,
- a reference to its `trackId`,
- an ordered list of `nodeId`s representing the Node graph,
- input configuration (Track input sources, audio/MIDI),
- output configuration (main output, bus routing, sidechain sends),
- fader state (gain, pan, balance),
- meter configuration (peak, RMS, LUFS, analysis parameters).

The Channel domain governs channel-level configuration.  
Node operations are handled in the Node domain.

---

## 3. Commands (Aura → Pulse)

### 3.1 Creation and Removal

Channels are created/removed via Track domain commands.  
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
configuration (LaneStreamNodes + fader/pan node).

Pulse then emits:

- `channel.graphReset`,  
- followed by Node-domain events describing the newly created Node graph.

---

## 4. Events (Pulse → Aura)

### 4.1 Structural Events

**`channel.created`**  
A Channel has been created due to Track enabling.

Includes:

- `channelId`,
- `trackId`,
- initial node list.

**`channel.deleted`**  
The Channel has been removed due to Track disabling.

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
- associated `trackId`,
- complete ordered list of `nodeId`s,
- input/output configuration,
- fader state,
- meter configuration.

Node snapshot data is included in the Node domain.  
Channel snapshot data binds them together.

Snapshots are authoritative; Aura replaces all Channel UI state accordingly.

---

## 6. Realtime and Cohort Considerations

Channel domain operations are non-realtime.  
However, Channel changes may:

- require Node graph rebuilds in Signal,
- invalidate anticipative buffers,
- trigger cohort reassignment via Pulse → Signal updates.

Cohort behaviour is governed by the Processing Cohorts architecture document.

Aura never interacts with cohorts directly; Channel events may trigger UI cues
(e.g. meter resets, graph rebuild animations).
