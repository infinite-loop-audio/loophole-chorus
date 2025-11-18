# Advanced Clips Architecture

This document defines the **advanced clip model** for Loophole.  
It expands upon the foundational clip architecture (`09-clips.md`) by adding:

- warp & stretch behaviour  
- multi-lane clip capabilities  
- MIDI and audio editing models  
- slicing & transient detection  
- embedded device lanes  
- clip-level variants  
- clip containers & nested clips  
- quantisation & groove at clip scope  
- nondestructive editing layers  
- clip DSP (pre-track)  
- exporting & rendering interactions  

Clips are one of the most complex and powerful entities in Loophole.  
This document establishes their full functional scope so Pulse and Signal can support them without retrofits.

---

## 1. Conceptual Overview

A **Clip** in Loophole is:

- a **temporal container** on a Track’s timeline,
- which may include **multiple Lanes** of media:
  - MIDI  
  - Audio  
  - Device Lanes  
  - Automation Lanes (clip-local)
  - Future media types (video, spectral slices)
- and also carries:
  - warp markers  
  - internal variants  
  - quantisation state  
  - groove  
  - transient map  
  - clip-level DSP  
  - container children (optional)  

Pulse owns the clip model entirely.  
Signal only receives **sample-accurate media instructions** after Pulse resolves:

- tempo  
- warps  
- quantisation  
- stretch factors  
- playback state  

---

## 2. Clip Structure

A clip consists of:

```
struct Clip {
  clipId;
  laneIds[];           // references to child Lanes
  start;               // musical position
  length;              // in beats/ticks
  loopLength?;         // if looping in Launcher
  offset;              // playback start offset
  warpMap;
  transientMap;
  quantisation;
  grooveId?;
  containerChildren[];
  variants[];
  activeVariantId?;
  clipDspChain[];
  metadata;
}
```

### 2.1 Start / Length / Offset

- All clip timing is stored in **musical time** (beats/ticks).
- Pulse converts to samples when sending to Signal.
- Offset allows:
  - slip editing  
  - clip shifting without altering Lane content  

---

## 3. Lanes Inside Clips

Clips may contain **multiple Lanes**, each representing a distinct media layer.

### 3.1 Lane Types

- **MIDI Lane**
- **Audio Lane**
- **Device Lane** (embedded Node)
- **Automation Lane**
- **Spectral/Morph Lane (future)**
- **Video Lane (future)**

Clips do **not** own tracks, but they may own *embedded devices* via Device Lanes.

### 3.2 Routing

Lane output flows to:

- a Clip-local **LaneStream** node *or*
- the Track’s default input chain.

Pulse stores the routing metadata; Signal instantiates the LaneStream nodes accordingly.

---

## 4. Audio Clip Model

Audio clips use a dual representation:

- **source-time** (samples)
- **musical-time** (beats/ticks)

### 4.1 Warp Map

```
struct WarpPoint {
  srcSample: int;
  dstBeat: float;  // musical position
  mode: enum { linear, elastic, transientLock }
}
```

WarpPoints define the mapping between audio and music.

Supports:

- Ableton-style free warp  
- transient-anchored warps  
- stretch by section  
- anchors for drums vs melodic material  
- reversible warp editing  
- copy/paste warp maps  

### 4.2 Warp Editing Tools

- Insert warp marker  
- Move warp marker  
- Stretch region  
- Consolidate clip  
- Align to transient  
- Fit tempo to selection  
- Multi-clip warp editing  
- Protect transient markers (drums mode)  
- Time-stretch previews (Signal sends preview stream)  

### 4.3 Stretch Algorithms

Pulse stores:

- chosen stretch algorithm (per region or global)
- grain size (for granular modes)
- formant mode (TruePolyphony-style)
- quality settings (economy vs high-end)

Signal executes:

- windowed FFT stretch  
- granular stretch  
- spectral harmonic stretch  
- future: machine-learning time-stretch  

---

## 5. MIDI Clip Model

MIDI clip contains:

- notes (with per-note expression)
- CC automation  
- channel pressure  
- pitchbend  
- MPE  
- articulation tags  
- transforms (non-destructive)

### 5.1 Notes

```
struct Note {
  id;
  pitch;
  velocity;
  startBeat;
  durationBeats;
  perNoteAutomation[];
  articulation?;
}
```

### 5.2 Expression & MPE

Pulse stores full per-note curves:

- pressure  
- timbre  
- per-note pitchbend  

Signal receives converted **sample-domain envelopes**.

### 5.3 MIDI Editing Layers

Non-destructive stack:

- quantisation  
- groove  
- randomisation  
- legato  
- scale snapping  
- humanisation  
- articulation mapping  
- generative transforms (Composer)

These layers are *metadata*, not destructive edits.

---

## 6. Quantisation & Groove (Clip Scope)

Clips can override:

- quantise grid  
- quantise strength  
- groove template  
- groove intensity  
- swing amount  
- randomisation  

Pulse applies all of this in **musical time**, then resolves to samples.

This allows:

- global groove + clip groove simultaneously  
- clip-specific swing  
- crossfading between two groove maps  

---

## 7. Slicing & Transients

### 7.1 Transient Map

Pulse stores:

```
struct Transient {
  sampleIndex;
  beatPosition?;
  confidence;
}
```

Generated via:

- built-in detector  
- Composer inference  
- manual editing  

### 7.2 Slicing Modes

- Slice to MIDI  
- Slice to clip lanes  
- Slice to multiple Device Lanes  
- Hybrid slicing (audio + MIDI conversion)

---

## 8. Clip Containers & Nested Clips

A Clip may contain **child Clips**, allowing:

- Ableton-style “Group Clips”  
- Cubase-style “Parts”  
- OpenTL-style hierarchical clip editing  
- Multi-clip editing operations  
- Composite editing (audio + MIDI + automation tied into a single entity)

### 8.1 Container Structure

```
struct ClipContainer {
  children[]: ClipId[];
  boundingRegion;
  transform; // stretch/offset applied to children
}
```

### 8.2 Use Cases

- Build reusable riffs  
- Variation morphing  
- Multi-track pattern workflows  
- Clip Launcher “super-scenes”

---

## 9. Clip Variants

Clip variants work exactly as defined in the project version/variant model, but scoped to clips.

Each clip may have:

- different warp maps  
- different MIDI edits  
- different device-lane processors  
- different slicing patterns  
- different DSP  
- alternate transformations  

Switching variant affects only the clip, not the Track or Arrangement.

---

## 10. Clip-Level DSP

A clip can host DSP before hitting the Track’s graph:

- clip EQ  
- clip gain  
- clip spectral edit  
- clip transient designer  
- clip-level effects  
- “fades as DSP” (curves → DSP kernel)  

Stored as:

```
clip.clipDspChain[]: NodeId[]
```

Executed by Signal *before* Track-level processing.

---

## 11. Editing Operations

Supported operations include:

- slip edit  
- stretch edit  
- stretch from content  
- warp edit  
- split / join  
- duplicate / repeat  
- reverse / invert  
- consolidate  
- quantise / groove  
- merge multi-lane clips  
- bounce-in-place  
- convert to container  
- extract MIDI from audio  
- extract transients  
- create variants  
- switch variants  

Undo is fully integrated (see Undo architecture).

---

## 12. Interaction with Launcher

Launcher clips:

- reference arrangement clips  
- may override:
  - loop length  
  - launch quantise  
  - groove  
  - warp mode  
  - device lanes  
- may store Launcher-only variants  
- trigger scenes based on clip boundaries

Pulse handles all quantisation and trigger timing.

---

## 13. Interaction with Tempo & Timebase

See `14-timebase-tempo-and-groove.md`.  
Clips depend on:

- beat ↔ sample mapping  
- warp ↔ tempo mapping  
- stretch factor from tempo  
- groove offsets  
- time signature grid  

Pulse resolves final playback structure for Signal.

---

## 14. Interaction with Rendering

Rendering respects:

- clip offset  
- warp position  
- fade curves  
- clip DSP  
- quantisation flattening (optional)  
- transients & slicing  
- nested clip transforms  

Pulse sends Signal the **fully resolved render graph**.

---

## 15. Summary

The advanced clip architecture defines:

- multi-lane, multi-media clip structures,  
- audio warp and transient systems,  
- deep MIDI capabilities,  
- quantisation & groove layers,  
- clip variants,  
- clip-level DSP,  
- clip containers and nested clips,  
- a nondestructive editing model,  
- tight integration with timebase, Launcher, rendering, and undo.

This model is essential for modern production workflows and must be implemented
at the Pulse level with sample-perfect outputs for Signal.
