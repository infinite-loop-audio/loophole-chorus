# Channel Routing & Send/Return Architecture

This document defines the **channel routing model** in Loophole, with a
particular focus on **sends, returns and signal flow** between Channels.

It sits conceptually between:

- `08-tracks-lanes-and-roles.md` – which defines what a Channel is and how it relates to Tracks and Lanes.
- `11-node-graph.md` – which defines how Nodes implement processing within a Channel.
- `12-mixer-and-channel-architecture.md` – which describes the Mixer UI, faders as Nodes, and console semantics.
- `06-processing-cohorts-and-anticipative-rendering.md` – which describes how routing impacts cohort partitioning.
- Pulse and Signal IPC:
  - `docs/specs/ipc/pulse/channel.md`
  - `docs/specs/ipc/pulse/routing.md`
  - `docs/specs/ipc/pulse/node.md`
  - `docs/specs/ipc/pulse/hardware-io.md`
  - `docs/specs/ipc/signal/graph.md`

The aim is to provide a **clear, consistent mental model** for:

- how audio flows between Channels,
- how Sends and Returns are realised as Nodes,
- how routing interacts with cohorts and anticipative rendering,
- how routing behaviour is persisted and exposed through IPC.

---

# 1. Goals

Channel routing in Loophole must:

1. **Be explicit and inspectable**
   - No hidden “special-case” routing behaviour.
   - Everything is visible in the Channel graph and in routing views.

2. **Treat sends and inserts as the same kind of building block**
   - A “send” is just a Node in the Channel’s Node graph.
   - A “return” is just another Channel feeding downstream.

3. **Support flexible topologies**
   - Traditional insert chains and FX buses.
   - Parallel processing.
   - Sidechains.
   - Track-to-track routing (within safety constraints).
   - Future expansion to surround / object-based topologies.

4. **Remain cohort- and latency-aware**
   - Routing constraints affect which Channels can be anticipatively rendered.
   - Sidechains and feedback paths must be constrained to keep the engine stable.

5. **Persist robust identities**
   - Routing relationships must survive project refactors, renumbering, and reordering.
   - Send routes remain stable even when Channels are renamed, moved or duplicated.

---

# 2. Conceptual Model

## 2.1 Channels, Nodes and Routing

At the highest level:

- A **Channel**:
  - is an audio signal path endpoint (or intermediate bus),
  - owns a **NodeGraph** (list/graph of Nodes),
  - has zero or more **inputs** and exactly one **main output**.

- A **Route**:
  - is a conceptual link between:
    - a **source Channel main signal** (or a Node tap), and
    - a **destination Channel input**.
  - describes *where* audio should flow, not *how* it is processed.

- A **Node**:
  - is a processing element inside a Channel’s NodeGraph,
  - may tap or inject signal to implement routing (e.g. SendNode),
  - is responsible for actual DSP.

Routing is therefore **two-layered**:

1. **Graph-internal routing** via Nodes:
   - `SendNode` taps the signal at a specific point.
   - `FaderNode`, `PanNode`, dynamics Nodes, etc. shape the signal in place.

2. **Channel graph routing** between Channels:
   - Pulse maintains a routing graph of Channel relationships.
   - Signal instantiates appropriate input/output bindings between Channel processes.

## 2.2 Main Types of Routes

Conceptually we support:

- **Direct Channel-to-Channel routes**
  - e.g. Track → Group Bus → Master.

- **Send routes**
  - Source Channel taps at a `SendNode` and routes to a target Channel (often an FX Bus).

- **Sidechain routes**
  - Source Channel feeds a sidechain input of a Node on another Channel.

- **Hardware routes**
  - Channel outputs to hardware outputs.
  - Hardware inputs feed Channels (recording and live monitoring).

All of these are described in a common routing model, even if the implementation differs.

---

# 3. Send & Return Model

## 3.1 Sends as Nodes

Loophole does **not** treat sends as a separate console-only concept.

Instead:

- A **Send** is realised as a `SendNode` in the source Channel’s NodeGraph.
- A **Return** is simply the main input of the destination Channel.

### SendNode Responsibilities

A `SendNode`:

- taps the signal at its position in the Node list,
- applies:
  - send level,
  - optional send pan,
  - optional send mute / bypass,
  - optional pre/post processing toggles for channel-specific behaviour,
- sends a copy of the signal to the target Channel’s input bus.

Because it lives in the Node list:

- **Pre/Post fader** behaviour is purely positional:
  - Pre-fader send: `SendNode` before `FaderNode`.
  - Post-fader send: `SendNode` after `FaderNode`.

- Any processing before the `SendNode` affects the send (e.g. pre-EQ sends).

- Multiple `SendNode`s can exist anywhere in the chain, allowing:
  - different **tap points** to different FX Channels,
  - complex parallel routing inside a single Channel.

## 3.2 Return Channels

An FX Channel (or “Return”) is simply a Channel whose:

- **main input** is fed by one or more `SendNode` routes,
- **output** routes to another Channel (e.g. a group or Master).

There is no special “return” type in the engine — the concept is purely:

- **semantic** (how the Channel is labelled and displayed),
- **topological** (where it appears in routing graphs and the mixer).

## 3.3 Multiple Sends & Summing

When multiple `SendNode`s target the same destination Channel:

- the destination Channel receives the **sum** of all send inputs,
- internal summing is done with appropriate headroom,
- send-level metres can be displayed in Aura per send.

The routing model supports:

- multiple send paths from a single Channel to a target,
- multiple Channels feeding a single FX Channel,
- nested FX structures (FX of FX).

---

# 4. Routing Graph Model (Pulse)

## 4.1 Routing Entities

Pulse maintains an explicit routing graph:

```
Route {
  routeId;             // stable ID
  sourceChannelId;
  sourceKind;          // main | send | sidechain
  sourceNodeId?;       // for send/sidechain nodes
  destinationChannelId;
  destinationInputKind; // main | aux | sidechain(N) | monitor
  enabled;
  role;                // main, bus, fx, sidechain, monitor, hardware
}
```

This graph:

- is the authoritative description of Channel-to-Channel relationships,
- is versioned with the project,
- is used to drive:
  - Signal graph wiring,
  - Mixer routing views,
  - diagnostics (e.g. routing cycles).

Nodes such as `SendNode` have **pointers** into this routing graph, but they do not own it.

## 4.2 Topology Constraints

Pulse enforces basic constraints to avoid pathological graphs:

- **No infinite cycles**:
  - cycles involving full-band feedback must be explicitly allowed and tagged as risky,
  - safe constructs (e.g. delay-feedback within a Node) are allowed.

- **Cohort-friendly routing**:
  - some routing patterns constrain anticipative rendering (see §5).

- **Channel-type rules**:
  - Master Channels cannot route to other Channels (except monitor / print buses).
  - Hardware returns must honour hardware topology and channel count.

---

# 5. Routing & Processing Cohorts

Routing interacts closely with the anticipative rendering architecture.

## 5.1 Simple Cases

For simple linear chains:

- Track → Group → Master

Pulse can:

- assign Group and Master to anticipative cohorts,
- keep live Tracks on realtime cohorts if required,
- schedule long-latency processing ahead of time.

## 5.2 Sends and FX Channels

For send/return routing:

- FX Channels can often be pre-rendered if:
  - their inputs are deterministic,
  - the send levels are stable or slowly changing.

- If a `SendNode` taps a Channel that is itself in a live cohort:
  - Pulse may either:
    - bring the FX Channel into the live cohort, or
    - limit how far ahead anticipative rendering can run.

The routing graph therefore informs:

- **cohort assignment decisions**,
- which Channels must remain in live sync with user interaction.

## 5.3 Sidechains

Sidechains introduce time dependencies between Channels:

- Sidechain source → sidechain input of Node on another Channel.

Rules:

- If **Channel B** depends on a sidechain from **Channel A**:
  - B cannot be rendered “ahead” of A beyond what the sidechain window allows.
  - Pulse may co-locate A and B in the same cohort.

- For offline rendering:
  - sidechain dependencies are respected in processing order.

---

# 6. Sidechain Routing

Sidechains are modelled as a specialised form of route:

- `sourceKind = main` or `send`
- `destinationInputKind = sidechain(N)`

The destination Node:

- declares one or more **sidechain input ports**,
- Pulse ensures that routing is compatible with:
  - channel count,
  - sample rate,
  - latency behaviour of the Node.

Sidechain routing is visible in:

- routing inspector,
- node inspector for the destination Node (showing its sidechain sources),
- optional matrix view in the mixer.

---

# 7. Hardware Routing

Hardware routing is a special case of Channel routing, defined in more detail in:

- `13-media-architecture.md` (for offline media),
- `24-rendering-and-offline-processing.md` (for renders),
- `25-control-surfaces-and-assistive-hardware.md` (for control devices),
- Pulse/Signal hardware IPC specs.

For audio I/O:

- **Input routing**:
  - Hardware inputs → Input Channels → Tracks (for recording/monitoring),
  - may go via dedicated Input Channels with processing Nodes.

- **Output routing**:
  - Mix Channels → Output Channels → hardware outputs,
  - allows:
    - multiple monitor paths,
    - dedicated stem/bounce outputs,
    - headphone mixes.

These routes are stored in the same routing graph, with `role = hardware`.

---

# 8. Persistence & Identity

Routing must be robust under:

- Channel renaming,
- Channel reordering,
- Track hierarchy changes,
- Channel duplication,
- Project version branching.

Key points:

- `routeId` is a stable identifier.
- Source/destination Channels use their **Channel identities**, not positional indices.
- `SendNode` holds a reference to `routeId`, not a bare Channel ID.

When a Channel is duplicated:

- Pulse may:
  - duplicate selected routes (e.g. sends) with new `routeId`s, or
  - ask the user whether routing should be cloned or reset.

Composer can assist in:

- recognising common routing templates,
- suggesting routing patterns for new Tracks (e.g. “Drum group → Drum verb bus”).

---

# 9. UX Considerations (Aura)

The routing architecture is exposed through multiple views (detailed in `26-ux-and-visual-layer.md`), but the core expectations are:

- Each Channel shows:
  - input source (Track, bus, hardware),
  - main output destination,
  - list of active sends (backed by `SendNode`s),
  - meters for input, post-fader output, and per-send level if enabled.

- A **routing matrix** provides:
  - overview of Channel-to-Channel relationships,
  - creation and deletion of routes,
  - visibility of sidechain paths.

- Node inspector shows:
  - send Node parameters (level, pan, destination),
  - sidechain inputs for dynamics Nodes.

The UX always reflects the underlying explicit model: no hidden routing.

---

# 10. IPC Integration

The routing architecture is realised over IPC as:

- **Pulse Routing Domain** (`docs/specs/ipc/pulse/routing.md`)
  - creates/updates/removes routes,
  - enumerates routing topology,
  - updates send/return semantics.

- **Pulse Channel & Node Domains** (`channel.md`, `node.md`)
  - create Channels and their NodeGraphs,
  - attach `SendNode`s referencing route IDs.

- **Signal Graph Domain** (`docs/specs/ipc/signal/graph.md`)
  - instantiates connections between Channel processes based on routing graph,
  - applies updates incrementally as routes change.

Routing changes are **atomic at the Pulse level**:

- Aura issues a high-level command (e.g. “create send from A to FX1”),
- Pulse:
  - creates a `Route`,
  - creates and inserts a `SendNode` in Channel A,
  - updates Channel graph in Signal,
  - emits events so Aura can update the mixer view.

---

# 11. Future Extensions

The routing model is intentionally forward-compatible with:

- **Surround / spatial routing**
  - Channels with more than 2 channels,
  - object-based outputs to renderers (e.g. Atmos-style objects),
  - dynamic routing based on listener position.

- **Routing presets and macros**
  - snapshotting whole routing configurations,
  - applying templates (e.g. “orchestral template” bus structure),
  - morphing between routing states.

- **Dynamic routing under script control**
  - scripts that adjust routing in response to project events,
  - context-aware FX routing suggestions from Composer.

All of these can be expressed by:

- extending `Route.role` and `destinationInputKind`,
- adding new Node types where necessary,
- keeping the core model (Channels, Nodes, Routes) unchanged.

---

# 12. Summary

Channel routing in Loophole is:

- **explicit** – no hidden magic; all routes and sends are visible and persisted,
- **Node-based** – sends are first-class Nodes in the Channel graph,
- **cohort-aware** – routing constraints feed into anticipative processing decisions,
- **hardware-inclusive** – hardware I/O is just another routing role,
- **future-proof** – ready for surround, object-based audio and advanced live workflows.

This document provides the conceptual foundation that the mixer, routing UI,
Pulse routing IPC, and Signal graph wiring all share.
