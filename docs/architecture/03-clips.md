# Clips Architecture

This document defines the conceptual model for Clips in Loophole. It covers
their structure, timing, relationship to Tracks, their ownership of Lanes, and
how Pulse resolves Clip contents into active audio, MIDI and control streams
during playback.

This is an architectural document: it describes concepts, shapes and flow, not
implementation specifics or IPC syntax.

---

## Contents

- [1. Overview](#1-overview)
- [2. Clip Placement and Timing](#2-clip-placement-and-timing)
  - [2.1 Time Range](#21-time-range)
  - [2.2 Snap, Grid and Stretch](#22-snap-grid-and-stretch)
  - [2.3 Start/End Behaviour](#23-startend-behaviour)
- [3. Clip Identity and Structure](#3-clip-identity-and-structure)
  - [3.1 Core Fields](#31-core-fields)
  - [3.2 Editing State](#32-editing-state)
  - [3.3 Transient Clip IDs](#33-transient-clip-ids)
- [4. Lanes Inside Clips](#4-lanes-inside-clips)
  - [4.1 Lane Models in Clips](#41-lane-models-in-clips)
  - [4.2 Allowed Lane Types](#42-allowed-lane-types)
  - [4.3 Per-Clip Lane Routing (Audio)](#43-perclip-lane-routing-audio)
  - [4.4 Per-Clip Lane Routing (MIDI)](#44-perclip-lane-routing-midi)
- [5. Content in Lanes](#5-content-in-lanes)
  - [5.1 Audio Lanes](#51-audio-lanes)
  - [5.2 MIDI Lanes](#52-midi-lanes)
  - [5.3 Automation Lanes](#53-automation-lanes)
  - [5.4 Video Lanes (Future)](#54-video-lanes-future)
  - [5.5 Device Lanes (Future)](#55-device-lanes-future)
- [6. Interaction with Tracks](#6-interaction-with-tracks)
  - [6.1 Track Aggregation](#61-track-aggregation)
  - [6.2 Overlapping Clips](#62-overlapping-clips)
  - [6.3 Muting and Bypass](#63-muting-and-bypass)
- [7. Playback Resolution](#7-playback-resolution)
  - [7.1 Lane Activation](#71-lane-activation)
  - [7.2 Lane → Channel Routing](#72-lane--channel-routing)
  - [7.3 State Updates](#73-state-updates)
- [8. Clip Editing Rules](#8-clip-editing-rules)
  - [8.1 Splitting](#81-splitting)
  - [8.2 Consolidation](#82-consolidation)
  - [8.3 Cross-Clip Lane Consistency](#83-crossclip-lane-consistency)
- [9. Interaction with Composer (Metadata and Automation Mapping)](#9-interaction-with-composer)
- [10. Future Extensions](#10-future-extensions)

---

## 1. Overview

A Clip is a time-bounded container placed on a Track. Clips contain one or more
Lanes, and Lanes contain the actual musical or media content: audio regions, MIDI
notes, automation curves, or other future types.

In Loophole:

- Tracks do **not** directly own Lanes.
- Clips own Lanes.
- Tracks simply arrange Clips along the timeline.
- Pulse resolves, at any playback time, which Clip Lanes are active on each Track
  and how they feed the Track’s Channel.

This allows different Clips on a single Track to contain different lane
configurations, enabling hybrid and experimental workflows.

---

## 2. Clip Placement and Timing

### 2.1 Time Range

A Clip defines a strict time range on its Track:

- `start` — absolute timeline position,
- `end` — absolute timeline position (> start),
- `length` — derived from start/end.

Pulse uses the Clip’s time range to:

- determine which Clips are active at any given playhead position,
- resolve cross-fade, comping and overlap interactions,
- identify which lanes to feed into LaneStream nodes.

Time ranges use project time units (beats or seconds, depending on timebase).
Tempo Lane changes may affect actual wall-clock duration of beat-based Clips.

### 2.2 Snap, Grid and Stretch

Clips are inherently quantised by user preference:

- snap to grid divisions,
- snap to bar/beat,
- snap to other Clip boundaries,
- optional free movement when snap is disabled.

Time-stretching is a lane-level behaviour (audio/MIDI) but is represented at the
Clip level, e.g.:

- Clip has a time-stretch multiplier,
- Lanes inside the Clip interpret their content using the Clip’s stretch
  context.

### 2.3 Start/End Behaviour

Clip boundaries do not automatically imply fades or truncation rules; these are
lane-level concerns:

- Audio lanes may define fade-in/out at Clip edges.
- MIDI lanes may clip events to boundaries or extend held notes depending on
  preference.
- Automation lanes may clamp to edge values.

---

## 3. Clip Identity and Structure

### 3.1 Core Fields

A Clip in Pulse includes:

- `id` — unique identifier,
- `hostTrackId`,
- `start`, `end`,
- `lanes[]` — ordered, typed collection of Lanes,
- optional:
  - `name`,
  - `colour`,
  - `muted`,
  - `playbackConditions` (future: conditional clips, follow actions),
  - `loop` / `loopLength`.

### 3.2 Editing State

Clips track editing metadata such as:

- selected lane(s),
- last-used editing mode (draw, scissors, comp, etc.),
- per-lane editor state (zoom, transient detection markers, etc.).

This state lives purely in Pulse/Aura and does not affect rendering or the engine.

### 3.3 Transient Clip IDs

Pulse uses stable IDs for Clips, but long-running edits (slice, glue, compilation)
may introduce new transient Clip IDs. Aura should treat IDs as immutable
references but should not assume permanence beyond a single editing transaction.

---

## 4. Lanes Inside Clips

### 4.1 Lane Models in Clips

Each Clip contains a sequence of Lanes. These are:

- ordered (by user or by type),
- typed (audio, MIDI, automation, etc.),
- self-contained (content stored within lane),
- time-bounded to the Clip’s own range.

### 4.2 Allowed Lane Types

Pulse governs which Lane types are permitted inside Clips. As of this
architecture:

- **Audio lanes** — one or more per Clip.
- **MIDI lanes** — one or more per Clip.
- **Automation lanes** — typically targeting Track or Channel parameters.
- **Video lanes** — future.
- **Device lanes** — not Clip-local; global context only.
- **Tempo/Chord/Groove/Marker lanes** — global context only.

### 4.3 Per-Clip Lane Routing (Audio)

Audio lanes must route to a LaneStream node in the Track’s Channel chain.

Default behaviour:

- When a Track first receives audio in any Clip, Pulse ensures that the
  Channel has a LaneStream node at the top of its processor chain.
- The first audio Lane added to a Clip routes to this LaneStream node by
  default.
- If a second audio Lane is added to that Clip, it also routes to the same
  LaneStream by default. Aura presents an option in the UI to “split” this Lane
  to its own LaneStream node.
- Splitting creates a new LaneStream node in the Channel chain, and the chosen
  audio Lane’s output is routed to that new node.
- For subsequent Clips on the same Track, audio Lanes route to the first
  LaneStream by default. Audio Lanes expose a UI element (such as a dropdown)
  allowing users to route that Lane’s output to a different LaneStream node.

### 4.4 Per-Clip Lane Routing (MIDI)

MIDI lanes route to **instrument processors** in the Channel chain.

Default behaviour:

- When a Track first gains an instrument in its Channel chain, MIDI Lanes on
  Clips route to that instrument by default.
- If there are multiple instruments in the chain, MIDI Lanes default to the
  first instrument unless configured otherwise.

Aura exposes:

- a UI control on MIDI Lanes that shows which instrument(s) they feed,
- an option to add a new instrument to the Channel chain directly from a MIDI
  Lane and route that Lane to the new instrument.

This mirrors the audio lane behaviour:

- adding MIDI content does not create a new processor by itself, but the UI
  provides a discoverable path to create and route a new instrument.
- users can build sophisticated MIDI processing chains (MIDI FX before
  instruments, multiple instruments per Track, etc.) while retaining a clear
  serial processor model.

---

## 5. Content in Lanes

### 5.1 Audio Lanes

Audio Lanes contain:

- regions referencing audio sources (files, resampled buffers),
- fade-in/out curves,
- stretch/mode parameters (re-pitch, preserve formants, preserve transients),
- optional per-region metadata (gain, crossfade modes),
- editorial markers (transients, warp markers).

### 5.2 MIDI Lanes

MIDI Lanes contain:

- notes,
- CC/aftertouch,
- polyphonic pressure (MPE),
- pitch and modulation curves,
- fold/hide channel routing rules.

They may feed one or multiple instrument processors.

### 5.3 Automation Lanes

Automation Lanes contain curves targeting:

- parameters of processors on the Track’s Channel,
- parameters of other Channels (mounted automation),
- potential device controls (future).

Automation is evaluated at Clip scope and clipped or combined when Clips
overlap according to Pulse rules.

### 5.4 Video Lanes (Future)

Video Lanes will contain:

- references to video media,
- trim handles,
- frame interpolation,
- colour correction,
- metadata for film workflows.

### 5.5 Device Lanes (Future)

Device Lanes provide structured control for:

- hardware I/O,
- monitor controllers,
- external effects,
- devices requiring parameter automation.

Device Lanes are likely **global-only** and not Clip-scoped.

---

## 6. Interaction with Tracks

### 6.1 Track Aggregation

A Track’s apparent set of Lanes in Aura is an **aggregate view**:

- If Clips on a Track contain different lane types, the Track header may show
  a union of those types.
- Aura can display only the Lanes relevant to the visible Clip, or all logical
  lane types the Track has used.

### 6.2 Overlapping Clips

When two Clips overlap:

- The later Clip takes precedence for the overlapping time region.
- Pulse resolves active Lanes by time and order.
- Automation follows precedence rules independently, depending on target.

Cross-Clip editing (comping, layering) is defined in later revisions.

### 6.3 Muting and Bypass

Clips can be muted, disabling all lanes inside them. Track mute affects the
Track’s Channel, and therefore implicitly disables all Clips.

---

## 7. Playback Resolution

### 7.1 Lane Activation

At runtime, Pulse continuously computes:

- which Clip Lanes are active at the current playhead,
- which LaneStreams or instrument processors they should feed,
- what automation values apply at that moment.

### 7.2 Lane → Channel Routing

Pulse builds a stable Channel processor chain. Clip Lanes do *not* mutate the
processor chain; instead:

- audio Lanes are mapped to the configured LaneStream slots,
- MIDI Lanes are mapped to instrument processors,
- automation Lanes generate parameter control signals.

### 7.3 State Updates

During playback, Pulse issues state change events when:

- a Clip enters or exits play,
- Lanes change routing or active status,
- automation targets change value.

Signal receives these updates as engine graph parameter updates, not structural
rebuilds.

---

## 8. Clip Editing Rules

### 8.1 Splitting

Splitting a Clip yields two Clips:

- each with its own Lanes,
- lane contents are trimmed accordingly,
- Clip-local metadata propagates cleanly.

### 8.2 Consolidation

Consolidating (render-in-place or bounce) merges multiple Lanes into a new
Clip with a single audio Lane pointing to a new rendered buffer. Original Clips
may be muted or archived depending on user preferences.

### 8.3 Cross-Clip Lane Consistency

Pulse does not require Clips on a Track to share lane structures. Aura handles
projection of lane headers and user editing tools on a per-clip basis.

---

## 9. Interaction with Composer (Metadata and Automation Mapping)

Automation Lanes inside Clips target parameters via fully qualified IDs. When
Clips contain Lanes that automate plugin parameters, Pulse may use metadata
from Composer to help:

- interpret what parameter a given Lane was automating (via semantic roles),
- suggest remapping of automation when a plugin version or variant changes,
- identify equivalent parameters when Clips are moved between Tracks or
  plugin instances change.

Such suggestions are advisory: Pulse remains the authority. Composer data is
optional and cached locally; Loophole must always remain operational without it.

---

## 10. Future Extensions

The Clip model is intentionally flexible. Future extensions may include:

- conditional playback (“play only on 3rd loop”, “follow action”),
- multi-take Clip containers,
- comp lanes inside a Clip,
- per-clip routing to subgroup Channels,
- per-clip Lane processors,
- Clip-level expressions or modulation sources.

These will integrate cleanly with the existing Clip → Lane → Channel model.
