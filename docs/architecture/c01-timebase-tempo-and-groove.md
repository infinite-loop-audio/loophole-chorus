# Timebase, Tempo & Groove Architecture

This document defines the complete **timebase**, **tempo**, and **groove**
architecture for Loophole.  
These systems underpin *every* timeline-driven subsystem:

- Clip position & stretching  
- Transport behaviour  
- MIDI timing  
- Automation interpolation  
- Clip Launcher synchronisation  
- Rendering boundaries  
- Comping and editing actions  
- Arrangement versions and variants  
- Metronome and grid display  
- External sync (future)

It extends the high-level definitions in `01-overview.md` and the project
structure in `07-project-versions-and-variants.md`.

Pulse owns all timebase-related model state.  
Signal executes against this model in real time.

---

## 1. Goals

- Provide a precise, deterministic, sample-accurate representation of musical
  time and tempo.
- Support:
  - tempo ramps,
  - tempo jumps,
  - beat-based and time-based regions,
  - clip warping and stretching,
  - groove quantisation,
  - global & per-clip groove applications,
  - multi-arrangement tempo maps.
- Allow future integration with:
  - Composer (tempo inference, groove extraction),
  - external sync sources,
  - generative tools.
- Make representation predictable for:
  - transport,
  - editing tools,
  - rendering,
  - Signal’s anticipative engine.

---

## 2. Conceptual Model

Loophole uses a **three-layered time system**:

1. **Absolute Time**  
   Fractional seconds (float or fixed-point).  
   Used by rendering, anticipative graph scheduling, media decode, etc.

2. **Sample Time**  
   Integer sample index relative to session start.  
   Used by Signal’s DSP loop, plugin processing, latency compensation.

3. **Musical Time**  
   Bars / Beats / Ticks.  
   Used by clips, MIDI, grids, editing tools, automation, Launcher synchronisation.

Pulse maintains the mapping between all three.

---

## 3. Timebase

The **timebase** represents the global musical timing context for an arrangement.

Properties:

- `tpqn` — ticks per quarter note (default 960).
- `barLength` — beats per bar (often 4).
- `beatUnit` — denominator of the time signature (e.g. 4 for 4/4).
- `nativeRate` — the nominal BPM reference.

### 3.1 Time Signature Map

A time signature map is an ordered list of:

```
struct TimeSignaturePoint {
  barIndex: int;
  numerator: int;
  denominator: int;
}
```

- Defines bar boundaries.
- Editing actions:
  - Add signature change
  - Remove signature
  - Move signature
  - Infer bars from clip grid (future)

Aura displays the bar grid based on this map.

---

## 4. Tempo Map

The **tempo map** defines BPM over time.

Pulse stores it as a list of **tempo events**, each one defining BPM at a given
musical location.

```
struct TempoEvent {
  position: MusicalPosition; // bar.beat.tick
  bpm: float;
  curve: enum { step, linear, exponential };
}
```

### 4.1 Event Types

- **Step change**  
  Immediate tempo jump. Used for DJ-like or film-oriented changes.

- **Linear ramp**  
  Smooth tempo transitions (common in modern DAWs).

- **Exponential ramp**  
  Useful for expressive accelerando/ritardando.

### 4.2 Sample-Accurate Mapping

Pulse computes:

- beat ↔ sample  
- bar/beat ↔ sample  
- tempo ramp curve → cumulative sample position

Signal never calculates these curves — it receives resolved, sample-based
instructions from Pulse.

---

## 5. Conversions

Pulse exposes a deterministic conversion API internally:

- `beatsToSamples(beats)`  
- `samplesToBeats(samples)`  
- `musicalToSamples(bar, beat, tick)`  
- `samplesToMusical(sampleIndex)`  
- `timeToSamples(seconds)`  
- `samplesToTime(sampleIndex)`

These operations depend on:

- tempo map,  
- ramp curves,  
- time signature map,  
- audio device sample rate (Signal reports via IPC).

Signal uses sample-domain timing exclusively.

---

## 6. Groove System

Groove defines **timing offsets** applied to notes, audio slices, and automation
to achieve feel:

- swing
- shuffle
- MPC-style grooves
- extracted grooves from audio
- Composer-generated grooves (future)

### 6.1 Groove Template

A groove template defines:

```
struct GrooveTemplate {
  id;
  name;
  resolution; // e.g. 1/8, 1/16, 1/32
  amount; // 0–1
  offsets[]; // per-step timing offsets in ticks
  velocityOffsets[]; // optional
}
```

### 6.2 Groove Sources

1. **Built-in libraries**  
2. **User-defined templates**  
3. **Extractor** (audio → groove)  
4. **Composer** (AI-derived groove content)

### 6.3 Application Levels

Groove can be applied at:

- **Arrangement level** (global feel)  
- **Track level** (individual rhythmic personality)  
- **Clip level**  
- **Lane level**  
- **Event level** (MIDI notes, audio transient slices)

Pulse stores groove references in each applicable domain:

- `clip.grooveId`
- `track.grooveId`
- `arrangement.defaultGrooveId`

### 6.4 Groove Calculations

Pulse applies groove offsets to:

- MIDI timestamps before quantisation  
- audio transient slice boundaries  
- automation timing if groove-locked

Groove is **not** applied in sample space — it is applied in **musical time** and
converted by Pulse into sample offsets.

---

## 7. Interaction with Clips

### 7.1 Audio Clips

Audio clips rely on:

- tempo → determining stretch factor,
- warp markers → mapping audio-time to musical-time,
- beatgrid → for slicing and quantising transients.

Key features enabled:

- beatgrid-aligned slicing,
- audio warp tied to tempo ramps,
- elastic audio in musical time,
- multi-lane groove-aware clips.

### 7.2 MIDI Clips

MIDI clips rely on:

- beat position,
- groove application,
- quantisation,
- timebase for note divisions,
- tempo for playback speed of CC ramps.

Pulse resolves:

- note-on and note-off → sample positions
- per-note MPE curves → sample envelope
- CC automation → sample curves

---

## 8. Interaction with Clip Launcher

The Launcher must follow:

- arrangement tempo/ramps,
- global groove,
- quantisation grid,
- scene launch quantise (1 bar, ½, ¼, etc.)

Pulse transmits:

- musical-to-sample aligned trigger times,
- quantised scene transitions,
- tempo-aware stop/launch boundaries.

Signal executes sample-aligned playback.

---

## 9. Editing Tools

### 9.1 Timebase Manipulation

Tools enabled:

- insert/remove bars,
- move bar boundaries,
- shift downbeats,
- apply groove to audio-on-grid,
- tempo ramp painting,
- tap tempo into map,
- beat detection,
- “align to transient”.

### 9.2 Smart Quantisation

Pulse supports quantisation modes:

- strict
- proportional (“humanise”)
- groove-based
- randomised (consistent randomness seeds)

Quantisation must be non-destructive in concept:

- metadata stored separately from MIDI events until committed,
- reversible with undo,
- variant safe.

---

## 10. Renderer Interaction

The rendering system relies heavily on tempo & timebase:

- resolving offline render boundaries,
- mapping musical selection → sample range,
- compensating for tempo changes mid-render,
- knowing pre-roll and post-roll,
- exporting stems aligned to bars even across tempo ramps.

Pulse sends to Signal:

- `render.start(sampleStart)`
- `render.stop(sampleEnd)`
- sample-aligned tempo map segments if needed

Signal handles:

- sample scheduling,
- plugin state,
- latency.

---

## 11. Pulse & Signal IPC Boundaries

Pulse is responsible for:

- the canonical timebase,
- storing/editing tempo map,
- converting everything to sample positions.

Signal never manipulates tempo or musical time.  
It only receives:

- sample indices,
- graph scheduling timings,
- transport clock values.

Pulse sends:

- `signal.transport.seek(sampleIndex)`
- `signal.transport.start(sampleIndex)`
- segment tempo data if required for offline rendering

Signal sends back:

- position updates (sample/time),
- sync signals for UI (Pulse decimates).

---

## 12. Composer Integration

Future hooks include:

- tempo inference from audio,
- groove extraction:
  - align markers → offsets,
- intelligent groove suggestions (“match feel to this reference clip”),
- structural tempo mapping (“find tempo changes in performance”),
- per-arrangement groove fingerprints.

All Composer augmentation sits *above* the Pulse timebase model and writes to it.

---

## 13. Summary

This document defines the core **musical time** infrastructure:

- sample/time/musical triad,
- time signatures and tempo maps,
- deterministic beat/sample mapping,
- groove templates and application,
- interactions with MIDI, audio, rendering, Launcher, and editing tools,
- the Pulse/Signal boundary for time operations.

All higher-level editing systems depend on the structures here, and Pulse’s
internal data model for clips, automation, and scheduling must conform to this
timebase architecture.
