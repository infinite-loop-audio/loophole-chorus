# Tracks, Channels and Lanes

This document defines the conceptual structure of Tracks, Channels and Lanes in the Loophole architecture. Tracks provide the user-facing container for timeline editing. Channels represent audio signal flow inside the engine. Lanes represent typed content inside Clips (audio, MIDI, automation, video, etc).

These concepts must remain strictly separated. Tracks are editorial and organisational. Channels are engine-level signal processors. Lanes are per-Clip content streams.

---

# 1. Overview

Loophole uses a unified Track model: there is only one type of Track, capable of hosting Clips, nested Tracks, and (optionally) a Channel. Audio, MIDI, video and automation all exist as Lanes inside Clips. A Track may or may not have a Channel attached; this determines whether its Clips produce audio.

Tracks define editing. Channels define audio processing. Lanes define content.

---

# 2. Tracks

A Track is the user-facing container for Clips and editing operations. Tracks MAY contain:

- child Tracks (nesting),
- Clips,
- automation,
- Lane metadata,
- a Channel (optional).

Tracks provide:

- visual grouping,
- sequencing/editing interfaces,
- timeline structure,
- default routing behaviour for attached Channels.

Nesting a Track inside another Track implicitly creates folder-like behaviour, but without introducing a separate “folder track” type. A parent Track MAY or MAY NOT have a Channel. If it does, its Channel MAY receive LaneStream(s) derived from child Tracks.

Tracks do not perform audio processing. They describe structure only.

---

# 3. Channels

A Channel is an engine-level construct representing audio signal processing. Channels have a dual nature:

- **Channel Model (Pulse)**: Pulse owns Channel structure, configuration, node ordering, and routing metadata. Pulse constructs Channel definitions and sends them to Signal as graph instructions.
- **Channel Runtime (Signal)**: Signal owns Channel execution, real-time processing, parameter application, and automation staging. Signal instantiates Channels from Pulse's instructions and executes them in real-time.

Channels:

- are configured by Pulse (model layer),
- are executed by Signal (engine layer),
- are visualised by Aura (UI layer),
- belong optionally to a Track.

A Channel contains:

- zero or more nodes (plugins or DSP blocks),
- zero or more LaneStream nodes (derived from Lanes),
- routing metadata,
- parameter states,
- automation staging buffers.

Channels MUST be real-time safe. They perform no dynamic allocation during processing, no locking, and no non-deterministic operations.

Channels never directly reference Tracks or Clips; they receive resolved graph instructions from Pulse. Pulse sends Channel configuration to Signal, which instantiates and executes the Channel runtime.

---

# 4. Nodes

Nodes are plugins or DSP units inside a Channel. They MAY include:

- virtual instruments,
- audio effects,
- MIDI nodes,
- utility or analysis nodes.

Nodes sit in a linear graph with strict ordering. A node MAY consume one or more LaneStreams as input. Serial mixing of LaneStreams is permitted and handled within the node graph.

Pulse is responsible for constructing and updating node graphs. Signal performs real-time execution.

---

# 5. Interaction with Composer (Metadata Only)

Tracks, Lanes and Channels do not depend directly on Composer, but Pulse MAY use Composer metadata when resolving node identities and parameter meanings during Track and Channel construction.

Composer provides:

- normalised plugin identity across formats and versions,
- logical (family-level) parameter identifiers,
- semantic roles (e.g. `filter.cutoff`, `fx.mix`, `amp.attack`),
- mapping suggestions when nodes are replaced or updated.

This metadata influences how Pulse interprets node and parameter structures but does NOT alter the Track, Lane or Channel architecture. Signal remains unaware of Composer entirely; it receives only finalised node and parameter instructions from Pulse.

---

# 6. Lanes

Lanes are typed content streams inside Clips. Every Clip contains one or more Lanes. Lane types include:

- audio,
- MIDI,
- automation,
- video,
- metadata (future),
- device control (future),
- global-only control lanes (e.g. tempo or groove lanes outside Tracks).

Lanes define the content, but not the DSP. Audio-producing Lanes route to LaneStreams inside a Channel. MIDI Lanes route to instruments defined in the Channel's node graph. Automation Lanes affect parameters in Pulse which translate into parameter updates for Signal.

A Track does not define Lanes directly. Clips define Lanes. Tracks constrain where Clips appear.

---

# 7. LaneStreams

A LaneStreamNode (also referred to as "LaneStream" for brevity) is a first-class Node type in the Channel's node graph. LaneStreamNodes are created by Pulse to receive audio from Clip audio-producing Lanes. One Track MAY have one or multiple LaneStreamNodes depending on how Clips are configured.

Default behaviour:

- The first audio Lane triggers creation of the first LaneStreamNode.
- Additional audio Lanes route to the same LaneStreamNode unless the user explicitly splits them.
- MIDI Lanes route to instrument nodes selected via the UI.

LaneStreamNodes appear in the node graph and MAY be reordered relative to other nodes, subject to constraints defined by Pulse.

Signal executes LaneStreamNodes as part of its processing graph. For details on LaneStreamNode as a Node type, see [Nodes Architecture](./09-nodes.md).

---

# 8. Track Nesting and Channel Behaviour

Tracks can be nested arbitrarily. Nesting affects:

- visual grouping,
- routing defaults,
- processing context.

If a parent Track has a Channel:

- its Channel MAY sum audio from child Tracks (if configured),
- the UI MAY show a LaneStream mixer view for its child content.

If a parent Track has no Channel:

- it acts purely as a structural folder,
- its children route elsewhere.

There is no dedicated “Group Track” type. Routing determines whether a Track behaves as a group, bus, folder, or hybrid.

---

# 9. Automation

Automation exists as Lanes within Clips. Each automation Lane refers to:

- a parameter (identified through Pulse),
- a target node (resolved by Pulse),
- time-varying values.

Automation MAY target:

- instrument parameters,
- effect parameters,
- Track-level parameters (gain, pan),
- global parameters (tempo, etc),
- virtual parameters created by Pulse.

Composer MAY provide metadata for automation mapping (e.g. semantic roles), but Pulse is solely responsible for resolving automation to real parameters.

Signal receives high-rate automation streams via real-time channels.

---

# 10. Global Lanes

Certain Lanes are global, not tied to a specific Track:

- Tempo
- Groove
- Time Signature
- Global Automation
- Device control lanes

Global Lanes follow the same Lane architecture but MUST appear only at the global timeline level.

Tracks cannot host global lanes.

---

# 11. Summary

- Tracks contain Clips and define structure.
- Channels contain nodes and define audio processing.
- Lanes define Clip content.
- LaneStreams bridge Lanes to Channel nodes.
- Nesting Tracks provides powerful structural organisation.
- Composer influences metadata and mapping, not architecture.
- Pulse resolves everything into a deterministic engine graph for Signal.
