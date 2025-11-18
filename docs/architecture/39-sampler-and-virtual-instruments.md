# Sampler & Virtual Instruments Architecture

This document defines the architecture for **samplers and virtual instruments**
in Loophole. It covers:

- how instruments integrate as Nodes and Channels,
- the Sampler as a first-class device,
- multi-sample, multi-layer instruments,
- per-note expression and MPE,
- integration with Clips, Lanes and Editors,
- future extensibility for advanced instrument types.

It complements:

- `06-tracks-channels-and-lanes.md`
- `07-clips.md`
- `08-parameters.md`
- `09-node-graph.md`
- `12-mixer-and-channel-architecture.md`
- `18-midi-architecture.md`
- `22-editor-architecture.md`
- `13-media-architecture.md`
- `33-plugin-library-and-browser.md`

Samplers and instruments are implemented primarily as **Nodes** in the
Channel graph, but they have deeper structural support in Clips, MIDI routing,
and media management.

---

# 1. Goals

Loophole’s instrument architecture should:

1. **Unify plugin instruments and internal samplers**
   - Treated as Nodes with shared parameter semantics.
   - Instruments are interchangeable from the routing perspective.

2. **Support deep sampler capability**
   - Multi-sample, multi-layer, round-robin, velocity and expression layers.
   - Clip-integrated sample management and editing.

3. **Play well with Lanes and Clips**
   - Multiple MIDI and audio Lanes per Track.
   - LaneStreams as the routing abstraction.

4. **Support per-note expression**
   - MPE,
   - polyphonic aftertouch,
   - per-note automation lanes.

5. **Integrate with the Media Library**
   - Samples managed as first-class media.
   - Easy drag/drop from media browser into Sampler and Clips.

6. **Be future-proof**
   - Custom instrument formats,
   - integrated drum machine instruments,
   - hybrid sampler-synth designs.

---

# 2. Instrument Nodes

Instruments appear in the NodeGraph as **InstrumentNodes**. Conceptually:

```
InstrumentNode {
  nodeId;
  kind: "plugin" | "sampler" | "internalInstrument";
  input: MidiInputRouting;
  output: AudioOutputRouting;
  parameters: ParameterSet;
  voices: VoiceStateSummary?;
}
```

- Plugin instruments: implemented via plugin lifecycle.
- Sampler: implemented as an internal instrument.
- Internal instruments: built-in synths and tools (future).

InstrumentNodes typically:

- sit near the start of the Channel graph,
- receive MIDI from one or more Lanes,
- output audio to be processed by downstream Nodes.

---

# 3. MIDI & Lane Integration

From `18-midi-architecture.md` and `06-tracks-channels-and-lanes.md`:

- A Track can have multiple MIDI Lanes.
- Each Lane can route to:
  - a specific InstrumentNode,
  - or be shared (one Lane → multiple instrument targets).

Routing is handled by:

- Lane routing metadata in Pulse,
- Pulse→Signal NodeGraph configuration.

Per-Lane routing supports:

- multi-layered instruments (e.g. two instruments stacked),
- complex “split/layer” key zones,
- per-Lane effects pre- or post-instrument.

---

# 4. Sampler Architecture

The **Sampler** is a core internal instrument type with deeper model support
than a simple plugin.

## 4.1 Conceptual Model

A Sampler instrument contains:

- one or more **SampleGroups**,
- each group contains **Regions** (zones),
- each region maps:
  - one or more samples (round-robin, multi-mic),
  - to a key/velocity range and conditions,
  - and an articulation / role (e.g. “snare centre”, “snare rim”, “ghost note”).

Simplified:

```
SamplerInstrument {
  programs: Program[];
}

Program {
  programId;
  name;
  groups: SampleGroup[];
}

SampleGroup {
  groupId;
  name;
  regions: SampleRegion[];
}

SampleRegion {
  regionId;
  samples: SampleRef[];
  keyRange;
  velocityRange;
  conditions?: RegionConditions;
  gain;
  tuning;
  envelopes;
  filters;
  modulation;
}
```

Media references point into the Media Library (`13-media-architecture.md`).

## 4.2 Sample Management

Sampler must:

- gracefully handle offline/remote media (Dropbox, external drives),
- interact with Media Library:
  - audition samples,
  - drag/drop into regions,
  - tag samples for better search.

Pulse:

- stores sampler programmes as part of the Node’s persistent state,
- ensures sample paths are relocatable via Media Library semantics.

---

# 5. Drum Kits & Drum-Focused Instruments

Drum workflows are first-class:

- Sampler supports dedicated **DrumKit mode**:
  - per-pad mapping (e.g. per MIDI note),
  - independent envelopes and FX per pad,
  - integration with Drum Sequencer editor.

Drum instruments can be:

- Sampler programmes specialised for drum use,
- third-party drum plugins integrated as InstrumentNodes,
- a hybrid drum-device UI inside Aura, backed by NodeGraph.

The Drum Sequencer:

- knows about kit layout (names, icons, colours),
- pulls metadata from the Sampler or plugin when available,
- shows pads and lanes aligned with the current instrument.

---

# 6. Per-Note Expression & MPE

Loophole supports:

- per-note pitch bend,
- per-note pressure,
- per-note control parameters (e.g. timbre),
- MPE.

Sampler and InstrumentNodes must:

- declare which forms of expression they support,
- map MIDI/MPE expression to internal parameters,
- expose mapping to Pulse for automation and editing.

Editors (Piano Roll, Expression Editor):

- display per-note expression lanes,
- allow editing per-note curves,
- integrate gestures (e.g. drawing vibrato).

---

# 7. Integration with Clips & Pipelines

## 7.1 Clip-Level Instrument Context

Each Clip can be associated with:

- the Track’s primary InstrumentNode,
- or may target a specific InstrumentNode in the Track’s NodeGraph.

This allows:

- per-Clip instrument switching,
- layered instruments per Clip (multiple Lanes → different Instruments).

## 7.2 Audio from Instruments

InstrumentNodes output audio that flows through Channel Nodes.

For offline/printing workflows:

- Instrument audio can be rendered to Clip Pipelines (see `20-rendering-and-offline-processing.md`),
- allowing:
  - in-place freeze,
  - destructive or non-destructive editing.

---

# 8. UI & Editor Integration

### 8.1 Sampler UI

Aura provides a rich Sampler editor UI:

- waveform views per sample region,
- zone editors (key, velocity, conditions),
- mapping displays (keyboard maps, pad grids),
- modulation matrix,
- per-region FX chains (implemented as internal or plugin Nodes).

All changes are:

- expressed as parameter/struct updates via Pulse,
- tracked in undo/redo history.

### 8.2 Instrument Browsers

The Plugin Library & Browser (`33-plugin-library-and-browser.md`) integrates:

- instrument tagging,
- multi-format instrument variants,
- Sampler programmes as loadable “instruments”,
- kit and preset selection.

### 8.3 Editor Interactions

Piano Roll and Drum Sequencer:

- Query instrument capabilities (note names, mapped drums, articulation IDs).
- Display note labels based on sampler groups/regions.
- Support “articulation lanes” (e.g. switching between staccato/legato articulations).

---

# 9. Performance & Cohorts

From `10-processing-cohorts-and-anticipative-rendering.md`:

- Instruments are often in the realtime cohort if:
  - they are non-deterministic,
  - they require live interaction (e.g. open UI, gestural control).

Sampler has to:

- be explicit about deterministic behaviour:
  - pure sample playback is deterministic,
  - random round-robin may be treated as non-deterministic unless seeded.
- integrate with anticipative processing:
  - pre-buffer instrument responses,
  - warm voices before clip start.

Pulse coordinates cohort placement based on:

- instrument type,
- plugin metadata (via Composer),
- project settings (strict determinism vs. performance).

---

# 10. IPC Considerations

At a high level:

### Aura → Pulse

- `instrument.loadProgram`
- `instrument.saveProgram`
- `sampler.addSampleRegion`
- `sampler.updateRegion`
- `sampler.removeRegion`
- `sampler.assignSample`
- `sampler.requestProgramList`
- `instrument.setExpressionMapping`

### Pulse → Aura

- `instrument.programList`
- `sampler.regionUpdated`
- `sampler.regionList`
- `sampler.sampleStatus` (online/offline/missing)
- `instrument.capabilities` (MPE, per-note expression, drums mapping)

### Pulse ↔ Signal

- Standard NodeGraph + parameter IPC.
- Bulk sampler programme state is serialised in plugin/internal instrument state blobs.
- Signal does not need to understand sampler structure; it only needs:
  - playback parameters,
  - sample references resolved by Pulse into streaming handles.

Exact message schemas live in `docs/specs/ipc/pulse/instrument.md` and related
files (to be defined/refined in detail as needed).

---

# 11. Future Instrument Types

This architecture supports:

- Hybrid sampler-synths (sample + oscillators).
- Granular/sample cloud instruments.
- Spectral resynthesis instruments.
- Multi-output drums with per-pad routing Nodes.
- Physical modelling instruments with per-voice state.

All via the same:

- InstrumentNode abstraction,
- Lane and Clip routing,
- parameter and expression mechanisms.

---

# 12. Summary

The Sampler & Virtual Instruments architecture:

- unifies internal and plugin-based instruments as Nodes,
- gives the Sampler deep model support for sophisticated workflows,
- tightly integrates with Clips, Lanes and Editors,
- supports MPE and per-note expression throughout,
- plugs into the Media Library and Plugin Browser,
- respects cohort and performance constraints.

This provides a flexible, powerful foundation for instruments in Loophole, from
simple one-shot samplers to advanced, multi-layered performance setups.
