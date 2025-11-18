# Rendering & Offline Processing Architecture

This document defines the complete rendering architecture for Loophole across:

- realtime playback  
- online prerendering & anticipative processing  
- offline rendering (bounce)  
- freezing & precomputation  
- clip-level ghost renders  
- track-level renders  
- project/session-level rendering  
- Launcher-aware rendering  
- version/variant-aware rendering  

Rendering interacts deeply with:

- Pulse (graph construction, job orchestration, timebase resolution)  
- Signal (sample-domain DSP execution)  
- Editing pipelines (ClipEditPipeline, MidiEditPipeline)  
- Node Graph (11-node-graph.md)  
- Comping (19-comping-architecture.md)  
- Automation/Modulation (17-automation-and-modulation.md)  
- Media Architecture (13-media-architecture.md)  

Signal executes all heavy DSP.  
Pulse controls render orchestration, lineage, media creation and filesystem output.

---

## 1. Goals

Rendering must be:

- deterministic and bit-stable,  
- phase-coherent across multiple channels,  
- capable of anticipative rendering for ultra-low latency workflows,  
- fully integrated with clip/track editing pipelines,  
- compatible with multichannel, surround, and immersive routing,  
- fully portable across platforms,  
- safe under crashes (Signal isolated),  
- parallelisable in the future across machines or cores.

Rendering should also:

- preserve lineage (like editing architecture),  
- never silently lose material,  
- allow reversible frozen tracks,  
- enable rapid bounce workflows.

---

## 2. Rendering Types

Loophole supports several rendering modes:

1. **Realtime playback**  
   - High responsiveness, hybrid realtime engine.

2. **Anticipative prerendering**  
   - Non-realtime background rendering of future segments.

3. **Clip ghost rendering**  
   - Evaluation of ClipEditPipelines (audio and MIDI).

4. **Track freezing**  
   - Long-term cache of a track’s rendered audio, replacing its graph temporarily.

5. **Bounce in place**  
   - Rendering a clip or track to a new media asset in the project.

6. **Stems export**  
   - Multi-track coordinated render.

7. **Full mixdown**  
   - Offline render of the whole timeline.

8. **Launcher-scene export**  
   - Export scenes linearly or as loops.

---

## 3. Pulse–Signal Rendering Contract

Rendering is orchestrated entirely by **Pulse**:

- Pulse constructs the render graph  
- Pulse resolves musical time → sample time  
- Pulse handles lineage and media creation  
- Pulse schedules jobs and retry logic  
- Pulse restarts Signal if needed  

Signal is a stateless execution engine:

- receives a complete DSP graph  
- receives all automation envelopes  
- receives modulator streams  
- receives ghost MIDI or flattened MIDI  
- renders sample-domain blocks  
- returns rendered buffers  

Signal **never** reads from disk directly; Pulse provides all media buffers.

---

## 4. Render Graph

Pulse builds a **RenderGraph**, a sample-domain representation of the Node Graph:

```
RenderGraph {
  nodes[];         // RenderNodes (post-validation)
  channels[];      // output channel configs
  automationCurves[];
  modulationStreams[];
  renderMode;      // realtime / offline / anticipative
  timeSpan;        // sample range
}
```

### 4.1 RenderNode

```
RenderNode {
  nodeId;
  nodeKind;
  resolvedParams;      // sample-domain parameter buffers
  resolvedMedia;       // buffers (audio / MIDI / ghost renders)
  inputs[];
  outputs[];
  runtimeFlags;        // oversampling, zero-latency, dry-run support
}
```

RenderGraph is:

- acyclic (Pulse enforces no cycles),
- validated before Signal execution,
- fully explicit (no hidden semantics).

---

## 5. Timeline Resolution

Pulse converts all musical-time constructs to sample space:

- tempo & timebase (`14-timebase-tempo-and-groove.md`)
- warp maps  
- groove offsets  
- clip offsets  
- comp boundaries  
- automation envelopes  
- MIDI events  
- crossfades  
- ARA regions  
- transient boundaries  

All resolved data is packaged into the RenderGraph.

---

## 6. Realtime Playback Rendering

Realtime playback uses the hybrid engine described in  
`06-processing-cohorts-and-anticipative-rendering.md`:

### 6.1 Realtime Cohort

- Contains nondeterministic or interaction-critical nodes:
  - active plugins with UIs  
  - send/return loops  
  - instruments with human input  
  - nodes marked “no prerender”  

### 6.2 Anticipative Cohort

- Deterministic nodes pre-rendered by Signal in large batches:
  - long FX chains  
  - heavy convolution  
  - CPU-intensive but stable nodes  
  - frozen tracks  

Pulse dynamically assigns nodes to cohorts based on:

- plugin metadata from Composer  
- user overrides  
- project performance context  
- past runtime diagnostics  

### 6.3 Block Processing

Realtime output is produced by:

- mixing anticipative prerender buffers  
- applying realtime cohort on top  
- scheduling automation envelopes  
- executing modulation streams  
- final post-fader output  

---

## 7. Clip Ghost Rendering

ClipEditPipeline (from `16-editing-and-nondestructive-layers.md`) is resolved by Pulse:

### 7.1 Ghost Render Conditions

Pulse ghost-renders a clip when:

- pipeline contains ARA  
- pipeline contains multiple clip-safe nodes  
- pipeline contains heavy transforms  
- warp changes require re-evaluation  
- groove quantisation changes  
- user requests consolidation  
- preparing for offline render  

### 7.2 Ghost Media

Ghost media functions like derived assets:

```
GhostMedia {
  mediaId;
  parentMediaId;
  lineage[];
  audioBuffers[];
  sampleRate;
  metadata;
}
```

---

## 8. Track Freezing

Track freezing is a specialised render mode:

### 8.1 Freeze Steps

1. Resolve track’s full render graph (excluding sends/returns if frozen muted).  
2. Render to a derived media asset.  
3. Replace track’s NodeGraph with a single **FrozenNode**:
   ```
   FrozenNode {
     mediaId;
     channelMode;
     latency;
     fadeBehaviour;
   }
   ```
4. Disable editing of:
   - NodeGraph  
   - Clip DSP  
   - automation (unless allowed in “post-freeze” mode)  

### 8.2 Unfreeze

- Ghost track state restored  
- NodeGraph reloaded  
- parameters recovered  
- automation lanes reattached  

Freeze is nondestructive and lineage-preserving.

---

## 9. Bounce in Place

Bounce-in-place produces new media assets:

- clip-level bounce  
- lane-level bounce  
- track-level bounce  
- selection bounce  
- comp-layer flattening  

Pulse creates:

- a new media asset in the pool  
- a new clip referencing that asset  
- lineage metadata  
- archive movement of original media (optional)

Bounce-in-place can optionally:

- include clip pipelines  
- include track processing  
- include sends/returns  
- include master bus (destructive flavour)

---

## 10. Stems Rendering

Pulse coordinates phase-aligned multi-track rendering:

- Each stem rendered independently but from a synced RenderGraph.  
- Parallelisable across multiple Signal instances or threads.  
- Shared timebase and cross-track automation.  
- Ghost MIDI and ARA resolved before stems export.

Stems may include:

- dry stems  
- wet stems  
- “pre-fader” stems  
- “post-fader” stems  
- grouped stems  
- surround/multichannel stems  

---

## 11. Full Mixdown Rendering

A complete offline render of the arrangement.

Process:

1. Pulse resolves full RenderGraph for timeline.  
2. Automation, modulation, comping, warp and groove are baked sample-accurately.  
3. Clip ghost-renders performed as needed.  
4. Multiple passes:
   - may run oversampled or in double/quad precision  
   - may include dithering  
   - optional realtime safety checks disabled  

5. Signal produces the full waveform.  
6. Pulse writes the file via MediaPool.  
7. Metadata embedded (markers, project info, Composer tags).

Formats supported:

- WAV, FLAC, AIFF, MP3 (optional module), Ogg  
- Multichannel formats for immersive  

---

## 12. Launcher Render Modes

Launcher scenes can be exported:

- as scene loops  
- as scene sequences  
- as linearised timeline exports  
- as arranged “performance renders”

Launcher automation and clip cycling are fully honoured.

Pulse builds a RenderGraph for the Launcher:

- scenes mapped to time segments  
- scene overlap rules  
- per-scene quantisation and offsets  

---

## 13. Rendering and ARA

ARA requires special handling:

- ARA steps must be resolved **before** building RenderGraph  
- If ARA regions are stale (post-comp edits), Pulse:
  - warns user  
  - invalidates regions  
  - offers re-analysis  
- During rendering, ARA-capable nodes are treated as deterministic processors  

ARA nodes may also require:

- pre-analysis passes  
- double-buffer render modes  
- pitch maps that persist across passes  

Pulse schedules these automatically.

---

## 14. Error Handling & Crash Safety

Signal is isolated:

- If Signal crashes during rendering:
  - Pulse restarts Signal  
  - resumes rendering from last checkpoint  
  - writes partial render state  
  - reports diagnostics  

- If a Node crashes:
  - Pulse quarantines the node  
  - retries in safe mode  
  - logs Composer telemetry  

---

## 15. Future: Distributed Rendering

Architecture is future-proof for:

- multiple Signal nodes  
- distributed render farm  
- local client + server render  
- network-based anticipative rendering  

Pulse would manage:

- job partitioning  
- node allocation  
- heartbeat checks  
- network recovery strategies  

The local architecture is already compatible.

---

## 16. Undo & Versions

Rendering operations affect:

- ClipEditPipeline (ghost updates)  
- frozen track states  
- media creation  
- bounce outputs  

Undo stacks track:

- freeze/unfreeze  
- bounce operations  
- render-related media lineage changes

Versions allow:

- alternate mixes  
- alternate freeze states  
- alternate render configurations

---

## 17. Summary

Loophole’s rendering architecture provides:

- unified realtime + offline design  
- anticipative processing for ultra-low latency  
- deterministic, sample-accurate rendering  
- clip ghost renders, track freezing, stems, bounce-in-place  
- full integration with ARA, comping, clip pipelines and node graph  
- flexible Launcher rendering  
- lineage-preserving media management  
- strong crash isolation  
- future support for distributed rendering  

This system ensures that rendering—from simple clip bounce to full project
mixdown—is stable, predictable, and extremely powerful.
