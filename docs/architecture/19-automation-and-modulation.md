# Automation & Modulation Architecture

This document defines the complete **automation and modulation model** for
Loophole, covering:

- automation lanes  
- parameter envelopes  
- modulation sources  
- routing of modulation to parameters  
- evaluation rules  
- sample-accurate rendering  
- UI semantics  
- interaction with Clips, Nodes, and the Clip/Media Editing pipelines  

It builds upon:

- `10-parameters.md`  
- `11-node-graph.md`  
- `12-mixer-and-channel-architecture.md`  
- `16-timebase-tempo-and-groove.md`  
- `17-advanced-clips.md`  
- `18-editing-and-nondestructive-layers.md`  

Pulse owns the canonical automation and modulation graph.  
Signal executes sample-accurate envelopes and modulations.

---

## 1. Goals

Automation and modulation must be:

- **high-resolution** and **sample-accurate**,  
- deterministic under all tempo/warp/groove conditions,  
- nondestructive (automation never replaces parameter defaults),  
- trackable across:
  - clips,  
  - track lanes,  
  - node graphs,  
  - variant/versions,  
- compatible with Clip Edit Pipelines and MIDI transforms,  
- highly expressive without clutter (UI must help, not hinder).

The model also must support:

- modulation stacking  
- param groups & linked automation  
- audio-rate modulation (within constraints)  
- envelope followers & LFOs  
- sidechain modulation  
- automation “takes” for comping  

---

## 2. Automation Domains

Automation exists at four main layers:

1. **Clip Automation**  
   Scoped to a clip; travels with the clip when moved/duplicated.

2. **Track Automation**  
   Scoped to a track; represents high-level mixing and performance moves.

3. **Lane Automation**  
   For clip lanes, LaneStreams, MIDI lanes, and audio lanes.

4. **Node Automation**  
   Automation of plugin or DSP parameters inside the Node graph.

All layers share a common representation, but operate at different scopes.

---

## 3. Parameter Binding

Every automatable parameter in Pulse has a unique, fully-qualified ParameterId:

```
<entityType>.<entityId>.param.<paramName>
```

Examples:

- `track.12.param.volume`
- `node.51.param.cutoff`
- `clip.9.lane.2.param.transientSensitivity`

Automation lanes bind directly to these ParameterIds.

### 3.1 Parameter Types

Automation supports:

- Continuous (float)
- Discrete (int)
- Boolean (mapped to stepped envelopes)
- Enum (stepped)
- Curve parameters (vector)
- Multi-parameter groups (via linked automation)

---

## 4. Automation Lanes

### 4.1 Structure

Each automation lane contains one envelope:

```
AutomationLane {
  automationId;
  parameterId;
  points[];        // list of AutomationPoint
  shapeDefault;    // default interpolation shape
  bounds;          // start/end in musical time
  muted: bool;
}
```

### 4.2 Points & Shapes

AutomationPoint:

```
AutomationPoint {
  id;
  positionBeat;      // position in beats/ticks
  value;
  shape: enum { step, linear, curve, bezier };
  shapeParams?;      // for curve/bezier shapes
}
```

Interpolation is performed in musical time, then converted to samples.

---

## 5. Automation Levels & Summation

Automation and modulation combine in the following order:

```
Base parameter value
  + Track automation
  + Node/Device automation
  + Clip automation
  + Modulation (sum of active sources)
  → Final parameter signal
```

If a parameter has a “normalised” span, the summation happens in normalised space, then mapped to the final domain.

---

## 6. Modulation System

Modulation is distinct from automation:

- **Automation** = timeline-driven, absolute or offset curves.  
- **Modulation** = live, dynamic signal that moves parameters.

### 6.1 Modulators

Modulators include:

- LFOs (bipolar or unipolar)
- Envelope followers
- MIDI-based modulation:
  - velocity  
  - note number  
  - channel pressure  
  - MPE dims
- Sidechain envelope followers
- Random/chaos sources
- Performer pads (future)
- Clip groove offsets (optional)

Each modulator is a Node in the modulation subsystem:

```
Modulator {
  modId;
  modType;
  parameters...;
}
```

### 6.2 Routing

Modulation Routing:

```
ModRoute {
  modId;
  paramId;
  amount;
  polarity;
  curve;
  minValue?;
  maxValue?;
}
```

Multiple modulators can target the same parameter.

Pulse stores and manages all routing.

Signal receives a “flattened” mod graph in sample space.

---

## 7. Evaluation Model

The evaluation pipeline for any parameter is:

1. Convert all automation to **sample-accurate envelopes**  
   - using tempo map  
   - with warp/groove  
   - converting shapes accordingly  

2. Evaluate all modulators as **sample streams**  
   - LFOs with deterministic phase  
   - envelope followers  
   - MIDI-driven signals  
   - sidechain analysis  

3. Multiply + sum modulation sources according to routing.  
4. Apply to parameter’s domain.  
5. Send the final sample-by-sample or block-by-block value to Signal’s plugin/Node parameter processor.

Pulse is the **authority for resolving musical-time automation**.  
Signal is the **executor of sample-domain signals**.

---

## 8. Clip Automation

Clip automation is:

- Stored inside the Clip (see `17-advanced-clips.md`)
- Follows the clip when moved, duplicated, stretched.
- Warp markers affect automation timing (automation must warp along with audio/MIDI).
- Groove and quantisation optionally apply to automation points.

Clip automation boundaries clamp outside-region evaluation.

---

## 9. Automation Comping

Automation comping works just like audio comping:

- automation “takes”  
- splice and blend curves  
- draw “comp curves”  
- flattening a comp produces a clean envelope

Pulse stores take layers:

```
AutomationCompLayer {
  layerId;
  points[];
  muted;
}
```

Comp result is maintained as a master envelope, with the takes preserved for undo/history.

---

## 10. Lane Automation

Lanes in a clip can carry automation for:

- audio properties (transient sensitivity, formant shift, per-lane gain)
- MIDI parameters (per-note micro parameters)
- device lanes (modulation processors embedded in the clip)

Automation is stored at lane scope and affects only that lane.

---

## 11. Node Automation

Nodes expose parameters affected by:

- clip automation  
- track automation  
- modulators  
- automation at the node’s own scope  

Pulse resolves all envelope and modulation results before sending sample-space parameter curves to Signal.

Automation is always applied *before* Clip DSP and Node graph.

---

## 12. Automation Editing Tools

Tools include:

- pencil  
- line tool  
- curve tool  
- shape presets  
- smoothing / thinning  
- tension control  
- randomisation  
- matching waveform transients (automation-from-audio)  
- groove-based automation quantise  
- gain-scaling of automation  
- multi-lane editing  

Editing tools are nondestructive and operate on automation points.

---

## 13. Interaction with Timebase

Automation points are always stored in **musical time**.

Pulse converts to samples using:

- tempo map  
- time signature  
- warp maps (for clip automation)  
- groove offsets  

This ensures:

- automation stays aligned with music even with complex tempo ramps  
- envelope shapes survive:
  - tempo edits  
  - clip stretching  
  - warp marker edits  

---

## 14. Interaction with Rendering

Pulse resolves all envelopes into **render envelopes** for offline render.

Signal receives:

- sample-domain automation envelopes  
- modulator sample streams  
- flattened modulation routing  
- any oversampled automation (for high-speed filters, etc.)  

Rendering maintains deterministic behaviour.

---

## 15. Interaction with Editing Layers

Automation and modulation interact with editing layers (see `18-editing-and-nondestructive-layers.md`) as follows:

- Clip pipelines may include *automation-driven* DSP; automation must update ghost renders when changed.
- Some modulator nodes may be Clip-Safe nodes and part of ClipEditPipeline.
- Automation of Clip DSP parameters works like Node automation but stays clip-scoped.
- Media-level editing does not carry automation.

---

## 16. Undo & Versions

Automation and modulation edits integrate with the history model:

- Adding/removing points  
- Moving/scaling points  
- Drawing curves  
- Adding/modifying modulators  
- Changing routing  

Each discrete action is undoable.

Versions/variants can:

- override automation  
- store different modulation routings  
- preserve automation takes  
- store alternate envelopes  

---

## 17. Summary

This architecture defines:

- a unified envelope + modulation model,
- sample-accurate automation,
- high-level modulation routing,
- clip/track/node-level automation layers,
- clip-aware warping & groove-aligned automation,
- automation comping,
- deep integration with the Node graph,
- consistent interactions with rendering & timebase,
- and a future-proof extension model (performer pads, audio-rate modulation, etc.).

Automation + modulation form a core creative system for Loophole and must be fully deterministic, high-resolution, and tightly integrated across Pulse, Signal, and Aura.
