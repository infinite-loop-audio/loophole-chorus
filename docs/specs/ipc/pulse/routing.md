# Pulse Routing Domain Specification

This document defines the Routing domain of the Pulse project protocol.  
It covers the commands issued by Aura to Pulse to configure the project-level
routing graph, and the events emitted by Pulse to notify Aura of routing
changes.

The **routing graph** connects:

- Channels to other Channels (busses, FX returns, submixes),
- Channels to hardware inputs and outputs,
- Channels to the project master output,
- Channels (or specific Nodes) to sidechain inputs.

Routing is maintained at the project level by Pulse.  
Signal receives a derived DSP routing graph that it realises internally.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Routing Concepts](#2-routing-concepts)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Bus and Master Configuration](#31-bus-and-master-configuration)  
  - [3.2 Channel–Channel Routing](#32-channelchannel-routing)  
  - [3.3 Sends and Returns](#33-sends-and-returns)  
  - [3.4 Sidechain Routing](#34-sidechain-routing)  
  - [3.5 Hardware I/O Mapping](#35-hardware-io-mapping)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Bus and Master Events](#41-bus-and-master-events)  
  - [4.2 Send and Return Events](#42-send-and-return-events)  
  - [4.3 Sidechain Events](#43-sidechain-events)  
  - [4.4 Hardware Mapping Events](#44-hardware-mapping-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)

---

## 1. Overview

Routing in Loophole is defined as a **project-level graph**:

- most Channels are Track-owned and represent Track signal paths,
- some Channels are standalone (busses, FX returns, master, I/O Channels),
- send connections between Channels are realised by SendNodes in the source Channel's Node graph,
- sidechains provide secondary inputs to Nodes,
- the master (or master chain) terminates the audible signal path.

The overall mixing and console architecture is described in
[Mixer & Channel Architecture](../../architecture/15-mixer-and-channel-architecture.md).

Routing commands express user intent around how Channels connect to each other
and to hardware. Pulse validates and applies these changes, then emits events
reflecting the resulting graph state. Pulse also coordinates with the Channel
and Node domains to ensure the DSP graph in Signal matches the routing model.

---

## 2. Routing Concepts

Key concepts in this domain:

- **Channel**  
  A DSP container with a `channelId`. Channels may be:
  - Track-owned, or
  - standalone with a role such as `master`, `bus`, `fx`, `send`, `return`, `io`.

- **Bus**  
  A standalone Channel used for submixing or FX returns.

- **Master**  
  One or more designated Channels forming the final output chain.

- **Send**  
  A connection from a source Channel to a target Channel (e.g. bus, FX Channel) realised by a SendNode in the source Channel's Node graph. The Routing domain describes the relationship between Channels and provides high-level controls; the actual send tap point and pre/post-fader behaviour are determined by the SendNode's position in the source Channel's Node list relative to the FaderNode, not via separate routing flags.

- **Sidechain**  
  A routing from a source Channel (or bus) into a Node’s sidechain input.

- **Hardware I/O Mapping**  
  Mapping of Channels to physical device inputs and outputs.

The Routing domain does not define the internal structure of Channels or Nodes.
Those are handled by their respective domains.

VCA-style control behaviour is expressed via parameter/control groups in the
Parameter domain, rather than as dedicated routing constructs.

---

## 3. Commands (Aura → Pulse)

Aura issues Routing commands when the user changes project routing.  
Pulse owns and enforces the routing graph.

### 3.1 Bus and Master Configuration

**`routing.createBus`**  
Create a standalone bus Channel (e.g. group or FX return). Pulse creates a
Channel with an appropriate role and returns/announces the new `channelId`.

**`routing.deleteBus`**  
Delete a bus Channel, subject to safety checks (e.g. no remaining required
dependencies).

**`routing.setMasterChannel`**  
Designate which Channel (or Channels, in multi-channel topologies) act as the
project master output path.

---

### 3.2 Channel–Channel Routing

**`routing.setChannelOutput`**  
Set the primary output target for a Channel.

Typical targets include:

- the master Channel,
- a bus Channel,
- a hardware output Channel.

This command complements Channel-level configuration and expresses intent at the
routing-graph level.

---

### 3.3 Sends and Returns

Sends represent additional connections from one Channel to another, realised in the DSP graph as SendNodes within the source Channel’s Node graph.

**`routing.addSend`**  
Create a send from a source Channel to a target Channel. Fields include:

- source channel,
- target channel,
- pre/post-fader mode (optional, defaults to post-fader),
- initial level and mute state.

Pulse creates a SendNode in the source Channel's Node graph at an appropriate position (post-fader by default, or pre-fader if specified) relative to the FaderNode, and wires its target Channel. Pulse allocates a `sendId` for routing-level tracking. Pre/post behaviour is a consequence of SendNode placement relative to FaderNode in the Node graph, not a separate routing flag.

**`routing.removeSend`**  
Remove a send identified by `sendId`. Pulse removes the corresponding SendNode from the source Channel’s Node graph.

**`routing.setSendLevel`**  
Update a send’s level or pan. This command proxies to the underlying SendNode’s parameters.

**`routing.setSendMode`**  
Change a send's mode (e.g. pre/post-fader). Pulse moves the SendNode to the appropriate position in the source Channel's Node list (before or after the FaderNode). This topology-based approach defines pre/post behaviour rather than using separate routing flags.

**`routing.setSendMuted`**  
Mute or unmute a send. This command proxies to the underlying SendNode’s enabled state or mute parameter.

Note: Advanced users or tools can manipulate SendNodes directly via the Node domain to position them at arbitrary points in the Channel graph (e.g. between specific plugin nodes) for more precise control.

---

### 3.4 Sidechain Routing

Sidechain routing can be expressed at routing level, independent of individual
Channel configuration.

**`routing.addSidechainRoute`**  
Create a sidechain routing from a source Channel to a Node’s sidechain input.

Pulse configures the underlying routing nodes and may coordinate with the
Channel and Node domains.

**`routing.removeSidechainRoute`**  
Remove a sidechain routing.

---

### 3.5 Hardware I/O Mapping

**`routing.setHardwareOutputMapping`**  
Map a Channel (typically the master or a bus) to one or more physical output
endpoints on the audio device.

**`routing.setHardwareInputMapping`**  
Map physical input endpoints to Channel or Track inputs.

Pulse ensures that:

- device capabilities are respected,
- conflicting mappings are handled according to project rules.

---

## 4. Events (Pulse → Aura)

Routing events are emitted by Pulse after valid command application or when
routing changes as a consequence of higher-level operations (e.g. loading a
project snapshot).

### 4.1 Bus and Master Events

**`routing.busCreated`**  
A new bus Channel has been created.

**`routing.busDeleted`**  
A bus Channel has been removed.

**`routing.masterChannelChanged`**  
The project master Channel configuration has changed.

---

### 4.2 Send and Return Events

Send events describe the lifecycle and configuration of send routes whose concrete implementation is SendNodes in the source Channel’s Node graph.

**`routing.sendAdded`**  
A send has been created between Channels. This event indicates that a SendNode has been created in the source Channel’s Node graph and configured to target the specified Channel.

**`routing.sendRemoved`**  
A send has been removed. This event indicates that the corresponding SendNode has been removed from the source Channel’s Node graph.

**`routing.sendUpdated`**  
Send configuration changed (level, mode, mute, pan). These changes are reflected in the underlying SendNode’s parameters or position in the Node graph.

---

### 4.3 Sidechain Events

**`routing.sidechainRouteAdded`**  
A sidechain route has been created.

**`routing.sidechainRouteRemoved`**  
A sidechain route has been removed.

---

### 4.4 Hardware Mapping Events

**`routing.hardwareOutputMappingChanged`**  
Hardware output routing updated.

**`routing.hardwareInputMappingChanged`**  
Hardware input mapping updated.

Aura can use these events to update routing visualisations and device I/O
panels.

---

## 5. Snapshot Semantics

Routing information is included in project-level snapshots. Snapshot entries 
include:

- bus Channels and their roles,
- master Channel designation,
- Channel output targets,
- sends (source, target, parameters) — these reflect the existence of SendNodes in the source Channels’ Node graphs,
- sidechain routes,
- hardware input/output mappings.

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

On snapshot application:

- Pulse replaces the current routing graph with the snapshot's routing graph.

Routing snapshots do not directly encode Channel or Node structure; those are
covered by the Channel and Node domains.

---

## 6. Realtime and Cohort Considerations

Routing changes are non-realtime operations. They may:

- trigger Channel and Node graph rebuilds,
- cause cohort reassignment in Signal,
- invalidate anticipative render buffers where signal flow changes.

Pulse ensures that routing changes are translated into safe graph updates for
Signal, respecting all realtime constraints. Aura never applies routing changes
directly to Signal; it only expresses intent via Routing domain commands.
