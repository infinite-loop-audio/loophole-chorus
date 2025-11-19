# Distributed Signal Cluster Architecture

## Contents

- [1. Overview](#1-overview)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [3. Roles and Topology](#3-roles-and-topology)
  - [3.1 Node Roles](#31-node-roles)
  - [3.2 Logical Topology](#32-logical-topology)
  - [3.3 Discovery and Registration](#33-discovery-and-registration)
- [4. Graph Partitioning and Scheduling](#4-graph-partitioning-and-scheduling)
  - [4.1 Partitionable vs Non-Partitionable Work](#41-partitionable-vs-non-partitionable-work)
  - [4.2 Subgraph Assignment](#42-subgraph-assignment)
  - [4.3 Scheduling Inputs](#43-scheduling-inputs)
- [5. Transport and Clock Synchronisation](#5-transport-and-clock-synchronisation)
- [6. Audio and Control Data Transport](#6-audio-and-control-data-transport)
  - [6.1 Audio Buffer Transport](#61-audio-buffer-transport)
  - [6.2 Parameter and Gesture Streams](#62-parameter-and-gesture-streams)
- [7. Plugin GUI Remoting](#7-plugin-gui-remoting)
  - [7.1 Rendering Model](#71-rendering-model)
  - [7.2 Input Model](#72-input-model)
  - [7.3 Performance Considerations](#73-performance-considerations)
- [8. Failure Handling and Degradation](#8-failure-handling-and-degradation)
- [9. Security and Trust Model](#9-security-and-trust-model)
- [10. Integration with Existing Architecture](#10-integration-with-existing-architecture)
  - [10.1 Pulse](#101-pulse)
  - [10.2 Signal](#102-signal)
  - [10.3 Aura](#103-aura)
  - [10.4 Composer](#104-composer)
- [11. User Experience](#11-user-experience)
- [12. Future Extensions](#12-future-extensions)
- [13. Constraints](#13-constraints)

---

## 1. Overview

The distributed Signal cluster architecture allows Loophole to run the audio
engine (**Signal**) across multiple machines and processes, while presenting a
single coherent engine to the user and to Pulse.

Key ideas:

- **Pulse** remains the global orchestration and state authority.
- **Signal** can run as multiple cooperating instances:
  - local host engine,
  - LAN “satellite” engines,
  - future cloud engines.
- The global processing graph is **partitioned into subgraphs** which are
  assigned to individual Signal nodes.
- Audio and control data are transported between nodes over the network with
  clear latency and determinism rules.
- Plugin GUIs can be **remoted** from remote Signal nodes back to Aura on the
  host machine.

The goal is to enable:

- transparent utilisation of additional machines on a local network,
- heavy anticipative or offline processing offloaded to remote nodes,
- a foundation for collaboration and cloud workflows.

---

## 2. Goals and Non-Goals

### Goals

- Allow multiple Signal instances to participate in a single Loophole session.
- Keep all **project state** and **graph structure** owned by Pulse.
- Maintain a **single logical transport** and timeline.
- Offload non-interactive, deterministic work to remote nodes without breaking
  real-time interaction.
- Support plugin GUI remoting from remote Signal instances to Aura.
- Degrade gracefully when remote nodes become unavailable.

### Non-Goals (initially)

- Arbitrary peer-to-peer topologies between Signal nodes (initial design uses a
  hub-and-spoke model).
- Hard real-time guarantees across WAN links (internet/cloud); initial focus is
  on LAN and offline/cloud rendering.
- Per-plugin user configuration of node placement (assignment is primarily
  automated by Pulse, with optional high-level controls later).

---

## 3. Roles and Topology

### 3.1 Node Roles

There are three conceptual roles in the cluster:

- **Host Pulse**  
  Single Pulse instance acting as orchestration authority and global model
  owner.

- **Host Signal**  
  The Signal instance running on the same machine as Pulse and Aura. This node:
  - handles all **latency-critical** and **interactive** paths,
  - is responsible for audio I/O to local hardware,
  - acts as the hub for other Signal nodes.

- **Satellite Signal Nodes**  
  Additional Signal instances running on other machines (or processes) that:
  - process assigned subgraphs,
  - return audio/control results to the host Signal,
  - may host plugin GUIs that are remoted to Aura.

In future, cloud engines are treated as a special class of satellite nodes with
different latency expectations and scheduling policies.

### 3.2 Logical Topology

Initial topology is **hub-and-spoke**:

- Pulse connects to **all** Signal nodes (host + satellites) using IPC/network
  channels.
- Host Signal acts as the **audio convergence point**:
  - all final audio is mixed here for local monitoring,
  - external audio hardware is attached here.
- Satellite Signal nodes connect to Pulse and (implicitly) to host Signal via
  defined audio/control channels.

Conceptually:

- Pulse sees a **single global processing graph**.
- The graph is partitioned into **subgraphs** assigned to nodes.
- Cross-node edges are realised as network audio/control connections.

### 3.3 Discovery and Registration

Signal nodes register with Pulse as follows:

- On start, a Signal node can:
  - advertise itself via LAN discovery (e.g. mDNS) or
  - be explicitly configured via address/port.
- The Pulse node maintains a **registry** of active Signal nodes, including:
  - node identifier,
  - network address,
  - hardware profile (CPU cores, memory, approximate performance),
  - supported plugin formats,
  - current load estimate.

Each Signal node has a **node ID** which is stable for the lifetime of a
session and is used in partition assignments and diagnostics.

---

## 4. Graph Partitioning and Scheduling

### 4.1 Partitionable vs Non-Partitionable Work

Pulse classifies graph elements according to whether they are safe to offload:

- **Non-partitionable / host-only:**
  - tracks with live audio inputs (record-armed, monitoring),
  - tracks with incoming MIDI from local devices,
  - nodes with open GUIs (for low-latency interaction),
  - any node or path marked as “real-time critical”,
  - control surface targets.

- **Partitionable / offloadable:**
  - deterministic processing without external time-varying inputs,
  - frozen or rendered tracks,
  - background bounces or stems,
  - long look-ahead processors that can tolerate higher buffer sizes,
  - post-fader and stem processing not feeding latency-critical outputs.

Determinism metadata and anticipative processing rules (defined elsewhere in
the architecture) are reused here.

### 4.2 Subgraph Assignment

Partitioning is done by Pulse:

1. Build the full processing graph (channels, nodes, routing).
2. Identify **cut points** where the graph can be split with minimal latency
   impact (e.g. between busses, after frozen tracks).
3. For each candidate subgraph:
   - estimate CPU cost,
   - estimate memory footprint,
   - determine latency sensitivity,
   - determine plugin availability (does the remote node support this plugin?).

4. Assign subgraphs to Signal nodes using a scheduling strategy, for example:
   - pack latency-critical segments on host,
   - spread heavy deterministic chains across satellites,
   - avoid splitting very short latency-sensitive chains.

Assignments are revisited periodically or when:

- new nodes appear,
- node performance changes significantly,
- user toggles “distribute processing” options.

### 4.3 Scheduling Inputs

For each processing block:

- Pulse sends **transport state** and **control updates** to all nodes.
- Each Signal node processes its assigned subgraph.
- Audio outputs from satellites are returned to host Signal and slotted into the
  global mix at the correct buffer boundaries.

Initial implementation assumes a fixed network latency budget for satellite
nodes and chooses buffer sizes accordingly. Later versions may adapt buffer
sizes per node.

---

## 5. Transport and Clock Synchronisation

All Signal nodes must share a **common notion of time** for:

- buffer alignment,
- modulation,
- tempo-synced effects,
- deterministic automation playback.

Pulse is the authoritative source of:

- transport state (play, stop, locate),
- project timebase (bars/beats, tempo map, time signatures),
- sample position (or equivalent high-precision timeline).

Signal nodes:

- compute their local processing schedule from the Pulse timebase,
- align block processing to Pulse buffer boundaries,
- never define their own independent transport.

Initial implementations assume:

- the host machine’s audio I/O is the primary **clock anchor**,
- satellites run slightly ahead in anticipative mode, buffered to account for
  network jitter and latency.

---

## 6. Audio and Control Data Transport

### 6.1 Audio Buffer Transport

Cross-node audio is transported as **buffered streams**:

- Each cross-node edge in the graph corresponds to:
  - one or more audio channels,
  - a buffer size,
  - a latency budget.

For each such connection:

- Satellite Signal nodes send audio buffers to host Signal over a network
  transport designed for low overhead and predictable behaviour.
- Host Signal inserts the buffers at the correct position in the mix.
- Pulse accounts for additional latency in its delay compensation model.

Initial transport design:

- use reliable, ordered transport for configuration and control messages,
- use an efficient, possibly framed binary protocol over TCP/QUIC for audio
  buffers,
- keep encoding as simple as possible (e.g. 32-bit float interleaved).

### 6.2 Parameter and Gesture Streams

Parameter automation and gestures are still driven by Pulse, but:

- satellites must receive the same automation envelopes as the host,
- gesture/value streams that target nodes on satellites are routed accordingly,
- high-frequency parameter streams are treated similarly to audio and must be
  bundled efficiently.

Gesture sessions:

- remain anchored in Pulse and Aura (as per the existing gesture/value-stream
  specification),
- carry a node ID / target node location so that Pulse can route the high-rate
  stream to the correct Signal node.

---

## 7. Plugin GUI Remoting

### 7.1 Rendering Model

When a plugin GUI is hosted on a remote Signal node, Aura must show that GUI as
if it were local.

High-level model:

- The remote Signal node:
  - hosts the actual plugin and its GUI,
  - renders the GUI into a framebuffer (or uses a platform-specific hook),
  - sends **frame updates** (full frames or dirty rectangles) to Aura.

- Aura:
  - displays the framebuffer inside an Electron window,
  - treats it like a normal plugin window from the user’s perspective.

Frame transport:

- Designed to be **good enough** for plugin GUIs:
  - modest resolution,
  - moderate frame rates,
  - focus on clarity and responsiveness.
- Implementation may evolve from:
  - simple full-frame bitmap streaming in the first iteration,
  - to region-based diffing and compression later.

### 7.2 Input Model

Aura captures user input for remote plugin windows:

- mouse movement, clicks, drags,
- keyboard events,
- focus/blur,
- window resize.

Events are forwarded to the hosting Signal node, which:

- injects them into the plugin GUI event loop,
- triggers redraws as needed,
- sends updated frames back to Aura.

Aura must keep the illusion that the plugin window is local:

- window management still follows the plugin UI architecture,
- screen-set awareness and positioning rules still apply,
- remote vs local hosting is mostly invisible to the user.

### 7.3 Performance Considerations

To keep remoted GUIs responsive:

- a plugin with an open GUI is strongly biased towards running on the **host
  Signal** where possible,
- only when the user explicitly opts in, or when performance demands it, should
  GUIs be remoted from satellites,
- frame rate and compression settings may be adaptive based on observed network
  performance.

---

## 8. Failure Handling and Degradation

When a satellite Signal node becomes unavailable (crash, network loss):

- Pulse detects missed heartbeats or failed RPCs.
- The affected subgraphs are:
  - muted and marked as degraded,
  - optionally reassigned to other nodes (including the host) if resources
    permit.

User feedback:

- Aura displays a **clear but non-intrusive** indication that some processing
  has gone offline,
- indicates which tracks/channels are affected,
- offers options:
  - retry connection,
  - reassign processing to host,
  - disable distributed processing temporarily.

Missing plugins on remote nodes are handled consistently with StackNodes and
plugin lifecycle rules.

---

## 9. Security and Trust Model

Signal nodes may run:

- on the same machine,
- on a trusted LAN,
- or (eventually) in cloud environments.

Minimum security requirements:

- **Authentication** between Pulse and Signal nodes (shared keys, certificates,
  or OS-level trust).
- **Encryption** for control and audio streams over untrusted networks.
- Authorisation / trust settings in Aura, so the user can control:
  - which machines are allowed to join a session,
  - whether cloud nodes are permitted,
  - which projects may use distributed nodes.

The initial implementation will assume a trusted LAN but should not preclude
later hardening.

---

## 10. Integration with Existing Architecture

### 10.1 Pulse

Pulse gains:

- a **cluster registry** of Signal nodes,
- a **partitioning and scheduling component** for assigning subgraphs to nodes,
- additional IPC domains for:
  - node registration and health,
  - subgraph deployment,
  - performance telemetry.

Pulse remains the **single source of truth** for:

- the project model,
- the global graph,
- determinism and anticipative processing policies.

### 10.2 Signal

Signal gains:

- the ability to operate as:
  - host engine,
  - satellite engine,
- a mechanism to:
  - receive subgraph definitions from Pulse,
  - process them in isolation,
  - report performance metrics back to Pulse,
  - optionally host plugin GUIs for remoting.

The distributed architecture should re-use as much of the existing graph and
node execution model as possible.

### 10.3 Aura

Aura is largely unaware of the distribution details. It:

- still talks to Pulse as the primary authority,
- still uses the same IPC envelope structure,
- receives enriched diagnostics and cluster state summaries for display,
- may show advanced views (per-node CPU, per-node assignment) in dedicated
  tooling panels.

For plugin GUIs, Aura must support both:

- local plugin windows,
- remoted plugin windows, using the same windowing and screen-set rules.

### 10.4 Composer

Composer can assist by:

- learning typical distribution patterns (e.g. which types of tracks benefit
  from offloading),
- helping suggest which tracks are safe to distribute,
- in future, guiding cloud/offline processing choices.

Composer does not participate directly in cluster control.

---

## 11. User Experience

Default behaviour:

- Distributed processing is **off by default** for reliability.
- When enabled, Loophole discovers available nodes and starts using them
  automatically, without requiring per-plugin configuration.
- The user sees:
  - a simple indicator that distributed processing is active,
  - optional high-level controls (e.g. “aggressive offload”, “prefer local”).

Advanced views (opt-in):

- show per-node CPU and memory,
- show which tracks/submixes are assigned to which nodes,
- allow power users to pin or exclude nodes for specific tasks.

At all times, Loophole must:

- preserve session integrity,
- avoid surprising destructive behaviour when nodes disappear,
- allow the user to disable distributed processing quickly if needed.

---

## 12. Future Extensions

Potential future additions built on this architecture:

- Cloud rendering services for stems, mixes, or AI-assisted processing.
- Collaboration sessions where different participants’ machines contribute
  processing as well as edits.
- More advanced graph partitioning (e.g. per-plugin or per-lane distribution).
- Hybrid scheduling that uses GPUs or dedicated DSP hardware alongside CPUs.
- Integration with experimental and AI-assisted subsystems described in the
  Future Systems documents.

---

## 13. Constraints

- The distributed architecture must **not** compromise the determinism and
  safety guarantees of the core engine.
- Latency-critical workflows must continue to function even with distributed
  processing disabled.
- The initial implementation should target LAN-based scenarios with modest
  network latency and jitter.
- Cloud and WAN use cases are considered **future extensions**, not part of the
  initial feature set.
- Complexity in Pulse and Signal must be contained; distribution is an
  extension of the existing execution model, not a separate code path.
