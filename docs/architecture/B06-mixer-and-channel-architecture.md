# Mixer & Channel Architecture

This document describes Loophole's mixing model and console architecture.
It explains how Tracks, Channels, Nodes, routing, and parameters combine to
form the mixer, and how Loophole deliberately diverges from traditional
analogue-console DAW designs (Cubase, Pro Tools, etc.).

The key goals are:

- one coherent internal model for all "faders" and "sends",
- minimal special cases,
- a clean separation between **audio graph** and **control relationships**,
- a console UI that feels familiar but is not constrained by old hardware
  metaphors.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Core Entities](#2-core-entities)  
  - [2.1 Tracks](#21-tracks)  
  - [2.2 Channels](#22-channels)  
  - [2.3 Nodes](#23-nodes)  
- [3. Faders as Nodes](#3-faders-as-nodes)  
  - [3.1 FaderNode](#31-fadernode)  
  - [3.2 PanNode and Gain Nodes](#32-pannode-and-gain-nodes)  
  - [3.3 Channel Roles, Same Fader Model](#33-channel-roles-same-fader-model)  
- [4. Sends and Returns as Nodes](#4-sends-and-returns-as-nodes)  
  - [4.1 SendNode](#41-sendnode)  
  - [4.2 Return Behaviour](#42-return-behaviour)  
  - [4.3 Pre/Post Fader as Topology](#43-prepost-fader-as-topology)  
- [5. MIDI and CC Levels](#5-midi-and-cc-levels)  
  - [5.1 No "MIDI Faders" in the Mixer](#51-no-midi-faders-in-the-mixer)  
  - [5.2 MIDI as Parameters and Lanes](#52-midi-as-parameters-and-lanes)  
- [6. Control Groups and VCA Behaviour](#6-control-groups-and-vca-behaviour)  
  - [6.1 Traditional VCA Faders](#61-traditional-vca-faders)  
  - [6.2 Control Groups as Parameter Groups](#62-control-groups-as-parameter-groups)  
  - [6.3 What Users See](#63-what-users-see)  
- [7. Folder Tracks and Grouping](#7-folder-tracks-and-grouping)  
- [8. Console UI Implications](#8-console-ui-implications)  
  - [8.1 Default Mixer View](#81-default-mixer-view)  
  - [8.2 Advanced Graph View](#82-advanced-graph-view)  
- [9. Relationship to Other Architecture Docs](#9-relationship-to-other-architecture-docs)
- [10. Routing Architecture](#10-routing-architecture)
  - [10.1 Overview](#101-overview)
  - [10.2 Routing Concepts](#102-routing-concepts)
  - [10.3 Channel-to-Channel Routing](#103-channel-to-channel-routing)
  - [10.4 Sends and Returns](#104-sends-and-returns)
  - [10.5 Sidechain Routing](#105-sidechain-routing)
  - [10.6 Hardware I/O Mapping](#106-hardware-io-mapping)
  - [10.7 Routing Graph and Project Structure](#107-routing-graph-and-project-structure)

---

## 1. Overview

Loophole's mixer is built on top of:

- the **Track model** (arrangement structure),
- the **Channel and Node graph** (DSP structure),
- the **Routing model** (project-level connections),
- the **Parameter and Automation model** (control and modulation).

Unlike many DAWs, Loophole does not distinguish "inserts" and "sends" as
separate console entities. Instead:

- everything in the signal path is a **Node**,
- sends are implemented as **SendNodes**,
- faders are implemented as **FaderNodes**,
- pre/post behaviour is defined by the **Node graph topology**.

Control relationships such as VCA-style master faders are expressed in the
**Parameter domain** as parameter groups, not as special Channel types.

---

## 2. Core Entities

### 2.1 Tracks

Tracks are the primary user-facing containers in the arrangement timeline.
They:

- own Clips and Lanes (see
  [Tracks, Lanes & Roles](./B01-tracks-lanes-and-roles.md)),
- may optionally own a Channel (via the Track domain),
- expose user-level flags (mute, solo, arm, monitor).

Tracks are part of the project model in Pulse. Tracks do not directly contain
DSP elements; that responsibility lies with Channels and Nodes.

### 2.2 Channels

Channels are DSP containers. Each Channel:

- has a `channelId`,
- may be **Track-owned** (typical audio/instrument Channel), or
- may be **standalone** with a role such as:

  - `master`,
  - `bus`,
  - `fx`,
  - `send`,
  - `return`,
  - `monitor`,
  - `io` (hardware I/O).

- owns an ordered list of Nodes forming its DSP graph,
- has input and output routing (see
  [Pulse Channel Domain Specification](../specs/ipc/pulse/channel.md) and
  [Pulse Routing Domain Specification](../specs/ipc/pulse/routing.md)).

Channels are the units that Signal realises as audio processing chains.

### 2.3 Nodes

Nodes are the fundamental DSP processing elements inside a Channel. Node types
include (non-exhaustively):

- **LaneStreamNode** (often shortened to 'LaneStream') – convert Clip Lane content into streams,
- **PluginNode** – wrap instrument/effect plugins (VST, CLAP, AU),
- **FaderNode** – implement fader gain,
- **PanNode** – implement panning/balance,
- **GainNode** – utility gain,
- **MixerNode** – merging or splitting channels,
- **SendNode** – send a tapped signal to another Channel,
- **AnalyzerNode** – provide metering and analysis,
- various routing and utility Nodes.

Nodes own parameters (Parameter domain) and are assigned to cohorts in the
[Processing Cohorts](./A06-processing-cohorts-and-anticipative-rendering.md) architecture.

For details, see the
[Pulse Node Domain Specification](../specs/ipc/pulse/node.md).

---

## 3. Faders as Nodes

### 3.1 FaderNode

In Loophole, a "fader" is not a special case; it is a built-in **FaderNode**
in the Channel's Node graph. The FaderNode:

- controls the final gain for the Channel (or a defined section of the graph),
- exposes a primary `gain` parameter,
- may have automation and gesture control applied to it,
- may be modulated by control groups (VCA-style behaviour),
- sits at a well-defined position in the Node list.

Typically, the FaderNode sits near the end of the Channel's graph, followed by
any final analysis nodes (meters) and output routing.

### 3.2 PanNode and Gain Nodes

Panning/balance is implemented as one or more **PanNodes** (and possibly
GainNodes) rather than as ad-hoc channel strip logic. A typical Channel chain
may look like:

- LaneStreamNode(s) (often shortened to 'LaneStream')  
- PluginNodes (instruments and effects)  
- optional utility GainNodes  
- PanNode  
- FaderNode  
- AnalyzerNode(s)  
- output routing  

This design allows full flexibility for reordering and advanced signal flows,
while still supporting a conventional mixer view.

### 3.3 Channel Roles, Same Fader Model

All Channels use the same FaderNode concept, regardless of role:

- Track Channels (audio/instrument),
- bus Channels,
- FX return Channels,
- master Channel(s),
- monitor Channels,
- hardware I/O Channels.

There are no separate "types of faders" at engine level.  
Differences appear as:

- Channel roles,
- UI grouping and presentation,
- parameter grouping (for VCA-like behaviour).

---

## 4. Sends and Returns as Nodes

### 4.1 SendNode

A send in Loophole is a **SendNode** in the Channel's Node graph.

A SendNode:

- taps the signal at its position in the Node list,
- forwards a copy of its input to a **target Channel** (usually a bus or FX),
- exposes parameters such as:

  - `sendLevel`,
  - `sendPan` (optional),
  - `muted` or `enabled`.

The SendNode normally passes its input through unchanged to the next Node, so
the original Channel path continues, but the tapped signal is also routed to
the target Channel.

### 4.2 Return Behaviour

"Returns" in classic DAWs appear as dedicated FX return channels.  
In Loophole, returns are simply Channels with appropriate roles and Node graphs.

A typical FX return setup:

- Source Channel: has one or more SendNodes targeting an FX bus Channel.
- FX bus Channel: contains Nodes implementing the FX chain (PluginNodes,
  GainNodes, FaderNode, etc.), then routes to another Channel (often the master).

If a distinct "return" concept is needed (for example, to model legacy
hardware workflows), it is expressed via Channel roles and routing, not via a
separate DSP primitive.

### 4.3 Pre/Post Fader as Topology

Traditional DAWs expose "pre/post-fader" send switches.  
In Loophole, pre/post behaviour is expressed purely by Node ordering:

- **pre-fader send**: SendNode placed before the FaderNode,
- **post-fader send**: SendNode placed after the FaderNode.

The same principle extends to "pre/post-compression", "pre/post-EQ", etc.:

- a SendNode inserted before a Node taps the signal *before* that processing,
- inserted after, taps it *after*.

High-level UI may still offer a "Pre/Post" toggle, but internally this is
implemented as moving/re-wiring the SendNode in the Node graph, not via
special-case audio engine flags.

---

## 5. MIDI and CC Levels

### 5.1 No "MIDI Faders" in the Mixer

Some DAWs (e.g. Cubase) expose "MIDI faders" for MIDI tracks, controlling:

- CC7 (MIDI volume),
- CC10 (MIDI pan),
- expression or other CCs.

This often leads to confusion, especially when tracks also produce audio.

In Loophole, the mixer does **not** provide separate "MIDI faders".  
The console focuses on **audio and routing**. MIDI control is modelled as
parameters and Lane content, not as separate fader types.

### 5.2 MIDI as Parameters and Lanes

MIDI-related levels and controls are handled via:

- MIDI Lanes in Clips (note velocity, events, etc.),
- parameter-based CCs (CC7, CC10, CC11, CC1, etc.),
- Automation targeting those parameters.

If a user wants a "track-level CC7 control", Aura may expose that as a
parameter tile in the mixer (not a fader), which:

- sends CC events via the Parameter domain,
- is routed to instruments as part of their parameter mapping,
- participates in automation like any other parameter.

The engine does not differentiate "MIDI faders" from other parameters.  
This avoids ambiguous semantics between audio level and MIDI CC level.

---

## 6. Control Groups and VCA Behaviour

### 6.1 Traditional VCA Faders

In traditional consoles/DAWs, VCA faders:

- do not carry audio themselves,
- control the gain of multiple other channels' faders as a group,
- allow relative gain changes with a single control.

Users often find VCAs confusing because:

- they look like channels but do not pass audio,
- they interact with automation in non-obvious ways.

### 6.2 Control Groups as Parameter Groups

In Loophole, VCA-style behaviour is modelled as **parameter grouping**, not as
a special Channel type.

A **Control Group**:

- is defined in the Parameter domain as a group of parameter targets
  (e.g. multiple FaderNode gain parameters, SendNode levels, etc.),
- has a group-level control that applies **relative changes** to the member
  parameters,
- may have its own automation,
- does not pass audio and is not implemented as a Channel or Node.

This design:

- separates control relationships from the audio graph,
- allows very general group definitions (e.g. a control group for send levels
  only, or for specific Sections of a mix),
- avoids the VCA naming confusion.

### 6.3 What Users See

In the UI, users might see:

- "Control Groups" or "Mix Groups" rather than "VCAs",
- each group with:

  - a name,
  - a "master" control (slider/rotary),
  - a list of member channels/parameters,
  - an automation lane.

Under the hood, this is implemented via parameter grouping and control curves.
The audio graph (Channels/Nodes) is unaffected; only parameters change.

---

## 7. Folder Tracks and Grouping

Folder tracks can optionally have Channels. This allows:

- pure organisation folders (no Channel),
- folder-as-bus scenarios (folder has Channel, children Tracks route to it).

This replaces concepts such as:

- "folder tracks with faders",
- "group channels hard-linked to arrangement folders".

Users can treat a folder Track with a Channel as a group bus; if no Channel is
attached, it is purely structural.

Because Channels are role-aware, a folder Channel may declare a role such as
`bus` or `group` and be presented differently in the mixer (colouring,
sectioning) without changing runtime semantics.

---

## 8. Console UI Implications

### 8.1 Default Mixer View

In its default form, the Loophole mixer shows:

- one strip per Channel (Track Channels, busses, master, FX, etc.),
- per-strip:

  - Meter (AnalyzerNodes),
  - Pan control (PanNode),
  - fader (FaderNode),
  - inserts (PluginNodes and other Nodes),
  - sends (SendNodes) in a familiar layout, even though internally they are
    Nodes in the same graph,
  - optional tiles for selected parameters (e.g. filter cutoff, CC controls).

Under the hood, there is no rigid distinction between "inserts" and "sends".
They are both Nodes; the UI simply chooses how to present them.

### 8.2 Advanced Graph View

An advanced editor may expose the Channel's Node graph directly:

- Nodes as graph elements,
- FaderNode, PanNode and SendNodes clearly visible,
- arbitrary routing between Nodes,
- ability to move SendNodes to any position (pre/post-fader, pre/post-EQ, etc.),
- ability to inspect per-Node cohort assignment and anticipative/live status.

This view is a natural consequence of treating everything as Nodes; there is no
need for separate routing editors.

---

## 9. Relationship to Other Architecture Docs

This document builds on and interacts with:

- [01-overview](./A01-overview.md) – high-level architecture overview,  
- [08-tracks-lanes-and-roles](./B01-tracks-lanes-and-roles.md) – structural
  definitions for Tracks and Channels,
- [09-clips](./B02-clips.md) – how Clips and Lanes feed into Channels,
- [10-parameters](./B03-parameters.md) – parameter identity, types and ownership,
- [06-processing-cohorts-and-anticipative-rendering](./A06-processing-cohorts-and-anticipative-rendering.md) –
  cohort assignment and anticipative rendering.

For IPC-level definitions:

- [Pulse Channel Domain Specification](../specs/ipc/pulse/channel.md) – channel
  configuration and lifecycle,
- [Pulse Node Domain Specification](../specs/ipc/pulse/node.md) – Node types and
  operations,
- [Pulse Routing Domain Specification](../specs/ipc/pulse/routing.md) – project
  routing graph and channel relationships.

The mixing model described here is the conceptual foundation those specs are
intended to follow. Any future changes to Channel or Node behaviour should be
reflected here first, then propagated to the IPC specifications.

---

## 10. Routing Architecture

### 10.1 Overview

Routing in Loophole is defined as a **project-level graph** that connects:

- Channels to other Channels (busses, FX returns, submixes),
- Channels to hardware inputs and outputs,
- Channels to the project master output,
- Channels (or specific Nodes) to sidechain inputs.

The routing graph is maintained at the project level by Pulse. Signal receives
a derived DSP routing graph that it realises internally. Routing commands
express user intent around how Channels connect to each other and to hardware.
Pulse validates and applies these changes, then emits events reflecting the
resulting graph state. Pulse also coordinates with the Channel and Node domains
to ensure the DSP graph in Signal matches the routing model.

### 10.2 Routing Concepts

Key concepts in routing:

- **Channel**  
  A DSP container with a `channelId`. Channels may be:
  - Track-owned, or
  - standalone with a role such as `master`, `bus`, `fx`, `send`, `return`, `io`.

- **Bus**  
  A standalone Channel used for submixing or FX returns.

- **Master**  
  One or more designated Channels forming the final output chain.

- **Send**  
  A connection from a source Channel to a target Channel (e.g. bus, FX Channel)
  realised by a SendNode in the source Channel's Node graph. The Routing domain
  describes the relationship between Channels and provides high-level controls;
  the actual send tap point and pre/post-fader behaviour are determined by the
  SendNode's position in the source Channel's Node list relative to the
  FaderNode, not via separate routing flags.

- **Sidechain**  
  A routing from a source Channel (or bus) into a Node's sidechain input.

- **Hardware I/O Mapping**  
  Mapping of Channels to physical device inputs and outputs.

The Routing domain does not define the internal structure of Channels or Nodes.
Those are handled by their respective domains.

VCA-style control behaviour is expressed via parameter/control groups in the
Parameter domain, rather than as dedicated routing constructs.

### 10.3 Channel-to-Channel Routing

Channels route to other Channels through their primary output target. Typical
targets include:

- the master Channel,
- a bus Channel,
- a hardware output Channel.

This routing is expressed at the routing-graph level and complements
Channel-level configuration. Pulse ensures that routing changes are translated
into safe graph updates for Signal, respecting all realtime constraints.

### 10.4 Sends and Returns

Sends represent additional connections from one Channel to another, realised in
the DSP graph as SendNodes within the source Channel's Node graph.

When a send is created:

- Pulse creates a SendNode in the source Channel's Node graph at an appropriate
  position (post-fader by default, or pre-fader if specified) relative to the
  FaderNode,
- the SendNode wires its target Channel,
- Pulse allocates a `sendId` for routing-level tracking.

Pre/post behaviour is a consequence of SendNode placement relative to
FaderNode in the Node graph, not a separate routing flag. Advanced users or
tools can manipulate SendNodes directly via the Node domain to position them at
arbitrary points in the Channel graph (e.g. between specific plugin nodes)
for more precise control.

Returns in classic DAWs appear as dedicated FX return channels. In Loophole,
returns are simply Channels with appropriate roles and Node graphs. A typical
FX return setup:

- Source Channel: has one or more SendNodes targeting an FX bus Channel.
- FX bus Channel: contains Nodes implementing the FX chain (PluginNodes,
  GainNodes, FaderNode, etc.), then routes to another Channel (often the master).

### 10.5 Sidechain Routing

Sidechain routing can be expressed at routing level, independent of individual
Channel configuration. Sidechain routes connect a source Channel to a Node's
sidechain input, allowing:

- dynamic processors (compressors, gates) to respond to external signals,
- creative routing scenarios where one Channel's audio modulates another's
  processing.

Pulse configures the underlying routing nodes and coordinates with the Channel
and Node domains to realise sidechain connections.

### 10.6 Hardware I/O Mapping

Hardware I/O mapping connects Channels to physical device inputs and outputs:

- **Output mapping**: Maps a Channel (typically the master or a bus) to one or
  more physical output endpoints on the audio device.
- **Input mapping**: Maps physical input endpoints to Channel or Track inputs.

Pulse ensures that:

- device capabilities are respected,
- conflicting mappings are handled according to project rules,
- routing changes do not disrupt realtime audio processing.

### 10.7 Routing Graph and Project Structure

The routing graph is part of the project structure and is included in
project-level snapshots. Snapshot entries include:

- bus Channels and their roles,
- master Channel designation,
- Channel output targets,
- sends (source, target, parameters) — these reflect the existence of SendNodes
  in the source Channels' Node graphs,
- sidechain routes,
- hardware input/output mappings.

Routing changes are non-realtime operations. They may:

- trigger Channel and Node graph rebuilds,
- cause cohort reassignment in Signal,
- invalidate anticipative render buffers where signal flow changes.

Pulse ensures that routing changes are translated into safe graph updates for
Signal, respecting all realtime constraints. Aura never applies routing
changes directly to Signal; it only expresses intent via Routing domain
commands.

