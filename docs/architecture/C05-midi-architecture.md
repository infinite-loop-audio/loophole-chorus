# MIDI Architecture

This document defines the full **MIDI and note-expression architecture** for
Loophole.  
It covers:

- MIDI event representation  
- MPE & per-note expression  
- clip-level and media-level MIDI  
- editing tools  
- nondestructive MIDI transform pipelines  
- interaction with groove, quantisation, warp and timebase  
- integration with Nodes, modulation, automation and rendering  
- performance considerations (ghost MIDI)  

This document builds on:

- `09-clips.md`
- `16-timebase-tempo-and-groove.md`
- `17-advanced-clips.md`
- `18-editing-and-nondestructive-layers.md`
- `19-automation-and-modulation.md`

Pulse owns all canonical MIDI data.  
Signal performs sample-accurate scheduling and modulation based on data
resolved by Pulse.

---

## 1. Goals

The MIDI system must:

- support modern performance standards (full MPE),
- be nondestructive by default, yet allow destructive-feeling edits,
- integrate with clip pipelines and automation,
- support MIDI processing Nodes,
- be sample-accurate and tempo/warp/groove-aware,
- enable high-level pattern editing tools,
- enable deep expression control (per-note and per-lane),
- integrate cleanly with the Clip Launcher,
- support rendering and offline bounce workflows.

---

## 2. MIDI Representation

Pulse stores MIDI using a structured event model that preserves:

- musical-time positions,
- per-note expression curves,
- deterministic ordering.

### 2.1 Note Event

```
struct Note {
  noteId;
  pitch;
  velocity;
  startBeat;         // musical time
  durationBeats;
  channel?;
  articulation?;
  expression: {
    pressureCurve?;
    timbreCurve?;
    pitchCurve?;
    customCurves[]?;
  }
}
```

### 2.2 Controller/Event Types

Pulse supports:

- **CC events** (continuous or stepped)
- **Pitch bend** (per-channel or per-note)
- **Channel pressure**
- **Polyphonic aftertouch**
- **Program changes**
- **Custom expression channels** (future)

### 2.3 MPE Compatibility

Loophole is fully MPE-capable:

- Per-note channel assignment (MPE zones)
- Independent pitch, timbre and pressure per note
- MPE curves stored as per-note automation envelopes

Pulse resolves MPE envelopes to sample-domain curves during playback/rendering.

---

## 3. MIDI in Clips

Every MIDI clip contains one or more MIDI lanes:

```
MidiLane {
  laneId;
  baseNotes[];             // untransformed core notes
  midiEditPipeline[];      // nondestructive transforms
  ghostMidi?;              // flattened note list for playback if needed
  metadata;
}
```

### 3.1 Clip Boundaries

- All note/event positions are stored in **musical time**.
- Stretching or warping a clip updates the rendering model (but not base notes).
- Duplicating or moving clips preserves notes with musical alignment.

### 3.2 Offset

Clip-level offset shifts MIDI playback relative to the clip’s left boundary.
This is a nondestructive transform.

---

## 4. MIDI Edit Pipeline

Defined in `18-editing-and-nondestructive-layers.md`, but expanded here.

### 4.1 Structure

```
MidiEditStep {
  id;
  kind: "hostOp" | "nodeOp" | "composerOp" | "future";
  nodeKind?;
  nodeConfig?;
  hostOpType?;
  hostOpParams?;
}
```

### 4.2 Transform Types

- Quantise (strict, proportional, groove-based)
- Humanise (timing, velocity)
- Scale snap
- Legato/overlap correction
- Randomise
- Velocity curves
- Articulation mapping
- Pattern transforms (invert, retrograde, rotate)
- Strum/human-style staggering
- Composition tools (Composer)
- MIDI effects (NodeOps)

Transform layers are stackable and nondestructive.

---

## 5. Ghost MIDI

When pipelines become complex or playback requires stability:

- Pulse materialises a **ghost MIDI** representation:
  - a flattened list of effective notes and controller streams.
- ghostMidi is stored alongside the lane:
  
```
ghostMidi {
  notes[];        // effective notes
  ccEvents[];
  pitchbend[];
  pressure[];
  metadata;
}
```

- It is invalidated when:
  - pipeline changes,
  - quantisation/groove changes,
  - warp changes.

During playback, Signal always receives **ghost or flattened MIDI** to guarantee:

- deterministic ordering,
- no expensive recomputation in real time.

---

## 6. Groove Integration

Groove (`16-timebase-tempo-and-groove.md`) applies to MIDI at:

- clip level,
- track level,
- arrangement level.

Groove offsets apply to:

- note onsets,
- note end times,
- optionally to CC ramp alignment.

Groove is applied within the MIDI Edit Pipeline before ghosting.

---

## 7. Automation & Modulation Interaction

MIDI parameters may be automated or modulated:

- per-note expression curves,
- pitchbend curves,
- clip-level “expression automation lanes”,
- MIDI effect Node parameters.

Pulse resolves:

- note-level automation (per-note envelopes),
- clip and track-level envelopes,
- modulation streams affecting MIDI Node parameters.

Signal receives:

- final note/envelope data,
- plus optional audio-rate modulation streams if relevant (e.g. MIDI LFO nodes).

---

## 8. Editing Tools

### 8.1 Piano Roll

Tools:

- draw/edit notes  
- resize  
- pitch drag  
- velocities  
- articulation editing  
- multi-select operations  
- scale tools  
- “magnetic scale”  
- fold/unfold lanes  
- key detection  
- ghost notes from other clips  
- time-stretching of MIDI  
- warp editing with shared anchors  

### 8.2 Expression Editing

- per-note expression lanes (pressure, timbre, per-note pitch, custom curves)
- multi-curve selection
- curve smoothing/tension/humanise
- expressive tools (slide, glide, vibrato shaping)

### 8.3 Global/Clip-Level Tools

- quantise  
- groove apply  
- randomise  
- MPE zone assignment  
- channel mapping  
- chord tools  
- arpeggiator transforms (host or Node-based)

---

## 9. MIDI Nodes in the Node Graph

Nodes can operate on MIDI:

- MIDI filter  
- MIDI note generator  
- MIDI LFO  
- chord generator  
- MIDI remapper  
- arpeggiator  
- MPE processor  
- MIDI → CV/Gate (future)

These are integrated via the same Node architecture (`11-node-graph.md`):

- MIDI-capable nodes can be used in Clip pipelines (if `clipSafe = true`).
- MIDI nodes can also appear in the track graph before instruments.

Pulse ensures the correct ordering of:

```
MidiEditPipeline → MIDI Nodes → InstrumentNode
```

Signal receives flattened MIDI data + Node graphs for sample-accurate execution.

---

## 10. Interaction with Timebase

All MIDI events are stored in musical time:

- pulses, beats, ticks  
- scaled by tempo  
- warped by clip warp-map  
- offset by clip offset  
- snapped via quantisation  
- shifted by groove

Pulse resolves these into final sample timings for:

- note on/off,
- per-note/MPE envelopes,
- CC streams,
- auto-generated envelopes from MIDI nodes.

---

## 11. Clip Launcher Integration

Launcher clips with MIDI:

- loop according to launcher loop settings,
- obey quantisation grid,
- adopt groove from clip or scene,
- send trigger times resolved by Pulse.

Launcher scenes may override:

- channel,
- articulation,
- velocity scaling,
- MPE behaviour,
- MIDI transforms (optional),
- instrument assignment (via track templates).

Pulse resolves Launcher-triggered MIDI into sample-accurate messages.

---

## 12. Rendering

Offline rendering requires:

- flattened note/event list (ghost or direct),
- lane-level MIDI,
- clip-level automation,
- track-level automation,
- instrument Node graph,
- modulation streams.

Pulse sends Signal:

- sample-resolved schedule of note events,
- sample-domain curves for MPE and CC,
- Node graph and parameter curves,
- any relevant modulation signals.

Signal executes complete audio rendering using the resulting note streams.

---

## 13. MIDI Media Assets

Imported `.mid` files are treated as **MIDI media assets**:

- stored in the media pool,
- may be edited at the media level (Media Editor),
- may produce derived MIDI assets,
- can be referenced by clips or used as composition sources.

This parallels audio media and shares lineage/archiving semantics (but without compression).

---

## 14. Undo & Versions

Pulse treats MIDI editing as atomic undoable operations:

- add/remove/move notes  
- transform layers  
- edit pipelines  
- MPE editing  
- quantise & groove operations  
- MIDI Node changes  

Versions/variants can:

- hold different pipelines,
- flatten ghost MIDI,
- switch between comped MIDI takes,
- store alternate expression curves.

---

## 15. Summary

The MIDI architecture defines:

- a detailed, structured event model,
- MPE and per-note expression as first-class citizens,
- nondestructive transform pipelines,
- ghost MIDI for performance and determinism,
- deep integration with groove, warp, automation and modulation,
- interaction with MIDI Nodes and Clip Pipelines,
- sample-accurate scheduling for rendering and playback,
- Launcher integration,
- media-level and clip-level workflows,
- version/variant compatibility.

This architecture fully unifies musical-time MIDI editing with Loophole’s
signal-based processing model and ensures maximum creative flexibility while
maintaining deterministic sample-level behaviour.
