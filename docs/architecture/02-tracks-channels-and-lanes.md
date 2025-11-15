# Tracks, Channels, Clips and Lanes

This document defines the conceptual model for Tracks, Channels, Clips and Lanes
in Loophole. It also describes how Lane audio and MIDI are presented in the
Channel processor chain in the form of LaneStream nodes and instruments.

This is an architectural document. It describes shapes and responsibilities, not
wire formats or specific APIs.

---

## Contents

- [1. Overview](#1-overview)
- [2. Channels](#2-channels)
- [3. Tracks and Clips](#3-tracks-and-clips)
  - [3.1 Tracks](#31-tracks)
  - [3.2 Clips](#32-clips)
- [4. Lanes](#4-lanes)
  - [4.1 Lane Types](#41-lane-types)
  - [4.2 Lane Scope](#42-lane-scope)
  - [4.3 Global Lanes](#43-global-lanes)
- [5. Channel Chains and LaneStreams](#5-channel-chains-and-lanestreams)
  - [5.1 Channel Processor Chain](#51-channel-processor-chain)
  - [5.2 Audio LaneStreams](#52-audio-lanestreams)
  - [5.3 MIDI Routing and Instruments](#53-midi-routing-and-instruments)
- [6. Nesting and Folder Behaviour](#6-nesting-and-folder-behaviour)
- [7. Automation and Device Lanes](#7-automation-and-device-lanes)
- [8. Views: Arrangement vs Mixer](#8-views-arrangement-vs-mixer)
- [9. Engine Integration](#9-engine-integration)

---

## 1. Overview

Loophole separates the concepts of:

- **Channels** — engine-level audio/MIDI processing units.
- **Tracks** — user-facing timeline containers for Clips.
- **Clips** — time-bounded containers for Lanes.
- **Lanes** — typed streams of musical or media data (audio, MIDI, tempo, etc).

There is a single conceptual Track type. Different behaviours (hybrid
instrument/audio, folder, bus) emerge from:

- whether a Track owns a Channel,
- which Clips exist on that Track,
- which Lanes those Clips contain,
- and how Lane outputs are routed into the Channel processor chain.

Global concerns such as tempo and groove, as well as potential device control,
are represented as Lanes with special placement rules rather than as separate
concepts.

Signal operates on Channels and processor chains. Pulse owns Tracks, Clips,
Lanes and Channels and derives engine graphs from them. Aura presents Tracks
and Lanes in the arrangement view and Channels in the mixer.

---

## 2. Channels

A **Channel** is an engine-level processing object. Channels exist only in Pulse
and Signal. They define:

- audio and MIDI inputs,
- a processor chain (instruments, effects, LaneStreams, etc.),
- routing destinations and sends,
- static properties (format, latency hints and other configuration).

Channels do not represent:

- clips or Lanes,
- track hierarchy or folder structure,
- visual layout or UI state.

Signal processes audio/MIDI purely in terms of Channels and their processor
chains.

---

## 3. Tracks and Clips

### 3.1 Tracks

A **Track** is a timeline container for Clips and a UI concept for sequencing,
editing and hierarchy. Each Track in Pulse includes:

- a unique identifier,
- an optional `channelId` (if it owns a Channel),
- track-level state flags (mute, solo, arm, monitor and other modes),
- a `parentTrackId` and `childTrackIds` for nesting,
- presentation properties (name, colour, icon, collapsed/expanded state).

Tracks do not directly contain Lanes. Instead, they contain Clips, and Clips
contain Lanes. Track “capabilities” (whether a Track behaves as audio, hybrid,
etc.) emerge from the Lanes present in its Clips.

### 3.2 Clips

A **Clip** is a time-bounded container hosted on a Track. It defines:

- a `hostTrackId`,
- a time range on the Track’s timeline,
- a collection of Lanes,
- lane-specific content (audio regions, MIDI notes, automation, etc.).

Clips determine what happens on a Track over time. Different Clips on the same
Track may contain different combinations or numbers of Lanes, subject to rules
imposed by Pulse.

---

## 4. Lanes

Lanes are the fundamental timeline entities in Loophole. They represent streams
of musical, media or control information.

### 4.1 Lane Types

Examples of Lane types include:

- Audio lanes (audio regions),
- MIDI lanes (note and controller data),
- Automation lanes,
- Video lanes (future),
- Tempo lanes,
- Chord lanes,
- Groove lanes,
- Marker lanes,
- Device lanes (future, for hardware and I/O control).

Lane type determines:

- what kind of content a Lane holds,
- how that content is interpreted,
- how it maps to Channels, devices or other targets.

### 4.2 Lane Scope

Lanes are scoped to Clips:

- Each Clip owns a set of Lanes.
- Clips on the same Track may contain different combinations of Lanes, within
  whatever constraints Pulse defines.
- At any given time position, Pulse resolves which Lanes from which Clips on a
  Track are active and how they contribute to the Track’s output.

From Aura’s perspective, Track lane headers and lane displays may project an
aggregate view of lanes used on that Track, while clip editors can show the
exact lanes for a specific Clip.

### 4.3 Global Lanes

Some Lane types represent global, project-wide structures (formerly referred to
as “Global Maps”). Examples include:

- Tempo lanes,
- Chord lanes,
- Groove lanes,
- Marker lanes,
- Potentially Device lanes.

These Lanes are not hosted by a Track. Instead, they are placed in a global
context and rendered by Aura in dedicated areas above or alongside the Track
area.

Global audio or video Lanes (e.g. reference audio or film master picture) may
also exist. They produce audio/video and route to dedicated Channels that are
not associated with Tracks.

Pulse defines which Lane types are permitted in global context and how they are
interpreted.

---

## 5. Channel Chains and LaneStreams

### 5.1 Channel Processor Chain

Each Channel has a processor chain, ordered from input to output. The chain can
include:

- LaneStream nodes (which pull audio from Lanes),
- MIDI processors,
- instruments,
- audio effects,
- other processing units implemented in Signal.

The processor chain is track-level and stable. It is not reconstructed at every
Clip boundary. Instead, Clips and Lanes determine what feeds LaneStreams and
instruments at any time.

This stability is important for real-time performance and plugin lifecycle
management.

### 5.2 Audio LaneStreams

Audio-producing Lanes from Clips on a Track feed into **LaneStream** nodes in
the Track’s Channel processor chain.

Behaviour:

- When audio is first added to a Clip on a Track that owns a Channel, Pulse
  ensures that the Channel has a LaneStream node at the top of its processor
  chain. This LaneStream node initially represents the Track’s “main” audio
  stream.
- The first audio Lane added to a Clip routes to this LaneStream by default.
- If a second audio Lane is added to that Clip, it also routes to the same
  LaneStream by default. Aura presents an option in the UI to “split” this Lane
  to its own LaneStream node.
- Splitting creates a new LaneStream node in the Channel chain, and the chosen
  audio Lane’s output is routed to that new node.
- Subsequent Clips on the same Track route their audio Lanes to the first
  LaneStream by default. Audio Lanes expose a UI element (such as a dropdown)
  allowing users to route that Lane’s output to a different LaneStream node.

LaneStream nodes appear in the Channel processor chain as units with basic mix
parameters (gain, pan and potentially phase and mute). They sum their input
audio into the running signal at their position in the chain.

Because LaneStream nodes are part of the processor chain, users can:

- interleave effects between different audio LaneStreams,
- control the order in which Lane contributions are added,
- build complex internal submix structures entirely within one Track.

### 5.3 MIDI Routing and Instruments

Instruments are hosted as processors in the Channel chain. There is no separate
“Instrument Lane” type:

- Instruments appear as processors in the chain (e.g. between MIDI processors
  and audio effects).
- MIDI Lanes in Clips produce MIDI data that is routed into instruments.

Default behaviour:

- When a Track first gains an instrument in its Channel chain, MIDI Lanes on
  Clips route to that instrument by default.
- If there are multiple instruments in the chain, MIDI Lanes default to the
  first instrument unless configured otherwise.

Aura exposes:

- a UI control on MIDI Lanes that shows which instrument(s) they feed,
- an option to add a new instrument to the Channel chain directly from a MIDI
  Lane and route that Lane to the new instrument.

This mirrors the audio Lane behaviour:

- Adding MIDI content does not create a new processor by itself, but the UI
  provides a discoverable path to create and route a new instrument.
- Users can build sophisticated MIDI processing chains (MIDI FX before
  instruments, multiple instruments per Track, etc.) while retaining a clear
  serial processor model.

---

## 6. Nesting and Folder Behaviour

Tracks may be nested:

- A Track with child Tracks behaves as a folder in the arrangement view.
- There is no separate “Folder Track” type; folder semantics arise from
  parent/child relationships.

Folder behaviour and bus behaviour depend on Channel ownership:

- A parent Track with no Channel is a pure container. Child Tracks route their
  Channels independently (e.g. to the master bus or other Channels).
- A parent Track with a Channel can act as a folder bus. Pulse may provide
  policies such as “auto-route children to this Track’s Channel” so that child
  outputs default to the parent Channel.

If a parent Track also has LaneStreams in its Channel chain, these can
represent:

- the parent’s own audio Lanes (if any),
- combined contributions from child Tracks (where configured),
- or more advanced internal routing patterns defined by Pulse.

---

## 7. Automation and Device Lanes

Automation is represented using automation Lanes:

- Each automation Lane belongs to a Clip and is associated with a host Track.
- An automation Lane targets a specific parameter, which may belong to:
  - the Track’s own Channel,
  - a plugin in that Channel’s chain,
  - another Channel,
  - or a device.

Pulse resolves automation Lanes into control signals applied to the appropriate
targets in the engine.

**Device Lanes** are a Lane type intended for controlling hardware and
device-level concepts, such as:

- audio interface channel trims,
- external hardware inserts,
- monitor controllers,
- external synthesizers or other devices.

Device Lanes are likely to exist primarily in global context (e.g. global
Device Lanes alongside tempo and marker Lanes), and may later be extended to
track-scoped use cases. Detailed semantics of Device Lanes are defined in
future specifications; this document records the concept at the architectural
level.

---

## 8. Views: Arrangement vs Mixer

Aura presents different views over the same underlying model.

**Arrangement view**:

- Shows Tracks and their Clips.
- Clips present their Lanes; aggregate lane headers may be used to summarise
  lanes present on a Track.
- Track nesting and folder behaviour are visible in this view.
- Channels are not displayed as standalone entities.

**Global lane area**:

- Shows Lanes that are not hosted by Tracks (global tempo, groove, markers,
  device Lanes and other global context).
- These Lanes influence playback and editing but are not part of the Track
  hierarchy.

**Mixer view**:

- Shows Channels and their processor chains.
- Tracks that own Channels label and colour the corresponding channel strips.
- Channels that are not associated with Tracks (returns, global reference
  Channels, device-related Channels) appear only in the mixer.
- LaneStream nodes and instruments appear in the processor list, with Lane
  routing configuration reflected in the UI.

---

## 9. Engine Integration

Signal operates purely on Channels and their processor chains. It does not know
about Tracks, Clips or Lanes as high-level concepts.

Pulse is responsible for:

- maintaining Tracks, Clips, Lanes and Channels,
- resolving which Clip Lanes are active at any time,
- determining how Lane audio and MIDI map into LaneStream nodes and instruments,
- constructing and updating the processor chains and routing for each Channel,
- issuing engine graph updates to Signal.

Aura interacts with Tracks, Clips and Lanes in the arrangement and with
Channels and processors in the mixer. It does not manipulate engine graphs
directly.
