# Editing & Nondestructive Layers Architecture

This document defines the **editing model** and **nondestructive layer stack**
for Loophole, across both audio and MIDI:

- how edits are represented,
- how they are applied,
- how they interact with media assets, clips, nodes, ARA, and rendering,
- how destructive-feeling workflows are built on top of nondestructive
  primitives.

It builds on:

- `07-project-versions-and-variants.md`
- `09-clips.md`
- `11-node-graph.md`
- `12-mixer-and-channel-architecture.md`
- `13-media-architecture.md`
- `16-timebase-tempo-and-groove.md`
- `17-advanced-clips.md`

Pulse owns the editing model and controls when/how operations are materialised
into rendered assets. Signal executes DSP and provides preview/processing
capabilities for heavy operations.

---

## 1. Goals

- Provide a unified model for **audio and MIDI editing** which:
  - never silently destroys original material,
  - supports both “destructive-feeling” and fully nondestructive workflows,
  - allows complex chains of transforms to be ordered and combined arbitrarily.
- Make editing operations **composable**:
  - host-native ops,
  - Node-based ops (reusing the Node interface),
  - plugin/ARA-powered ops.
- Allow Pulse to decide when to:
  - evaluate edits live,
  - or materialise them as **ghost renders** (derived media assets).
- Integrate cleanly with:
  - Nodes & routing,
  - ARA integration,
  - Undo & versions,
  - media archiving.

---

## 2. Layered Editing Model

Editing is layered into distinct levels:

1. **Media Level**  
   Operations that conceptually act on a *media asset* itself.

2. **Clip Level**  
   Operations that act on *one clip (or clip lane) in context*.

3. **Clip DSP Chain**  
   DSP node chain that follows clip playback but precedes track-level nodes.

4. **Track / Node Graph**  
   The main realtime signal graph (see `11-node-graph.md` and `12-mixer-and-channel-architecture.md`).

Each level can be implemented partly as:

- **Node-based transforms** (reusing Node semantics),
- **host-native edit primitives**,
- **plugin-driven transforms** (including ARA).

Pulse controls how these layers are evaluated and when they result in new
derived assets.

---

## 3. Clip Edit Pipelines

For each *audio lane within a clip*, Pulse maintains a **Clip Edit Pipeline**:

```
ClipEditPipeline = [
  ClipEditStep1,
  ClipEditStep2,
  ...
]
```

Each **ClipEditStep** is, conceptually:

```
struct ClipEditStep {
  id;
  kind: "hostOp" | "nodeOp" | "araOp" | "future";
  nodeKind?;     // for nodeOp: which Node type is used
  nodeConfig?;   // serialised Node parameters/settings
  hostOpType?;   // for hostOp
  hostOpParams?; // for hostOp
  araRegionId?;  // for araOp
  enabled: bool;
}
```

### 3.1 Node-Based Steps

Where possible, **ClipEditStep** uses the existing **Node interface**:

- `kind: "nodeOp"` indicates that a step is realised by a Node.
- `nodeKind` refers to a Node type defined in the Node architecture.
- `nodeConfig` represents that Node’s parameter state.

This allows us to reuse:

- existing DSP units,
- audio processing semantics,
- parameter handling,
- diagnostic and profiling infrastructure.

**Important constraint:**  
Not all Node types are valid in a ClipEditPipeline. For example:

- `SendNode`, `ReturnNode`, `BusNode` and other routing/topology nodes are
  *not* permitted as ClipEdit steps.
- Only Nodes that are:
  - **pure DSP transforms** (EQ, compression, saturation, etc.),
  - **time-domain transforms** (stretch, transient shaping, convolution, etc.),
  - or **analysis/edit transforms** (spectral edits, restorations)
  are allowed.

The valid subset is defined as a **Clip-Safe Node class**:

- `node.isClipSafe = true` → eligible for use in ClipEditPipeline.
- Pulse enforces this when building or editing pipelines.

### 3.2 Host-Native Steps

Some operations do not warrant a full Node, e.g.:

- trim/split/consolidate operations,
- metadata tagging,
- simple gain change baked into media metadata.

These are represented as `kind: "hostOp"` with structured parameters and are
evaluated by Pulse’s internal edit engine.

### 3.3 ARA-Based Steps

ARA integration (see `15-advanced-clips.md`) appears as:

- `kind: "araOp"`
- referencing one or more `AraRegion` entities.

In practice, ARA steps are realised by **ARA-capable PluginNodes** (see
`11-node-graph.md`), but the ClipEditPipeline only stores references to
regions and bindings. Signal receives sample-accurate region descriptions and
executes those ARA plugins in the correct order.

---

## 4. Evaluation & Ghost Renders

Pulse may evaluate ClipEditPipelines in two modes:

1. **Live Evaluation**  
   - For cheap steps (e.g. simple gain, fade curves) that can run in realtime
     without significant cost.
   - Implemented either as:
     - nodes inserted into the live lane graph, or
     - lightweight per-sample operations in Signal.

2. **Ghost Render (Materialised)**  
   - For heavy pipelines (complex spectral edits, stacked ARA, multiple
     Clip-Safe nodes).
   - Pulse requests a **render** of the lane with its pipeline applied.
   - Result: a new **derived media asset** (ghost render).
   - The clip then plays back the ghost render as its source, while Pulse
     retains the pipeline definition for future edits.

Ghost renders are:

- stored as normal media assets in the media pool,
- tagged as **derived** with lineage information,
- replaceable when the pipeline changes,
- discardable when no longer needed (re-renderable from originals).

---

## 5. Media-Level Editing & Asset Lineage

### 5.1 Media Editor (Destructive-Feeling)

The **Media Editor** operates on *media assets* rather than clips:

- Context: “edit this file at the media level”.
- Internally, editing uses the same primitives:
  - hostOps,
  - Clip-Safe nodeOps,
  - potentially ARA or other external editors.

However, when the user **commits** media-level changes:

- Pulse does **not** overwrite the asset bytes in place.
- Instead, it creates a **new derived media asset**:

  - `newMediaId` derives from `baseMediaId`,
  - asset lineage recorded (`parentMediaId`, reason: `"mediaEdit"`),
  - project references are rewired to point to `newMediaId` (where applicable).

The old media asset is:

- retained in the media pool,
- typically transitioned to an archived state (see below),
- hidden from normal UI views but still recoverable.

### 5.2 Media Archiving & Compression

When assets become obsolete (no longer used, or explicitly superseded):

- Pulse may transition them to an **archived** state and:

  - convert them to a lossless compressed format (e.g. FLAC),
  - move them into a project-level archive folder:

    - `Media/Archive/<mediaId>.flac`

  - update the media index with:
    - new path,
    - compression codec,
    - lineage metadata (`archivedFromMediaId`, reason, timestamp).

This allows Loophole to:

- keep a complete history of original recordings and important derived states,
- preserve space through compression,
- still offer advanced users and tools (including Composer) the ability to
  inspect and, if necessary, restore from archived assets.

A separate “Clean Up Archive” workflow will allow users to inspect and purge
archived assets explicitly.

---

## 6. MIDI Editing & Pipelines

The same architectural concepts apply to MIDI:

### 6.1 MIDI Base Data & Media

MIDI note data may originate from:

- imported MIDI files (treated as optional **MIDI media assets**),
- or directly from clip-local editing.

Pulse treats:

- the **base note set** as the starting point,
- a stack of **MIDI transforms** applied on top.

### 6.2 MIDI Edit Pipeline

For each MIDI clip (or lane), Pulse maintains a **MidiEditPipeline**:

```
MidiEditPipeline = [
  MidiEditStep1,
  MidiEditStep2,
  ...
]
```

Each **MidiEditStep** describes a transform, such as:

- quantise,
- humanise,
- scale snapping,
- articulation mapping,
- randomisation,
- pattern generation (Composer),
- plugin-based MIDI effect renders.

Internally, many transforms can be expressed in the same **Node** vocabulary
by using **MIDI-capable Nodes** (as defined in the Node architecture):

- `kind: "nodeOp"`, with `nodeKind` representing a MIDI Node.
- Alternatively, host-native transforms (`kind: "hostOp"`) can be used for
  simple or especially common operations.

### 6.3 Live vs Materialised MIDI

At playback time, Pulse may:

- apply MidiEditPipeline transforms on-the-fly for cheap operations, or
- materialise a **ghost MIDI** representation (flattened note events) when
  pipelines become complex or when deterministic caching is desirable.

Ghost MIDI is:

- a flattened, effective note set derived from the base + pipeline,
- used for performance/stability,
- re-materialised if the pipeline changes,
- serialised as part of the project representation.

---

## 7. ARA & External Editors in the Stack

ARA-based editing is modelled as steps in the ClipEditPipeline:

- `kind: "araOp"`, referencing one or more `AraRegion` objects.
- ARA-capable PluginNodes realise these steps in Signal.

The overall processing order (conceptual) for an audio lane is:

1. **Raw media asset** (possibly derived from earlier media-level edits)
2. **ClipEditPipeline** (hostOps + Clip-Safe Node Ops + ARA Ops)
3. **Clip DSP chain** (`clip.clipDspChain[]`, see `15-advanced-clips.md`)
4. **Track-level Node graph**

Media-level editing sits *upstream*:

- Media Editor → new derived media asset → used as input to clips.

Pulse may choose to:

- incorporate ClipEditPipeline into ghost renders,
- treat ARA steps as either live or baked depending on cost,
- manage dependencies between ARA, hostOps and NodeOps.

---

## 8. Undo, Versions & Variants

### 8.1 Undo

Each edit operation is represented in Pulse’s model as:

- changes to:
  - pipelines (add/remove/modify steps),
  - media asset lineage (new derived media, rewiring references),
  - ARA region bindings,
  - clip DSP chains.

Undo & History (see its dedicated architecture doc) will:

- treat groups of low-level edit modifications as higher-level undoable actions
  (e.g. “Edit Clip”, “Apply Media Edit”, “Update ARA Region”),
- preserve pipelines and derived asset references atomically so that undo/redo
  does not desynchronise media from edits.

### 8.2 Versions & Variants

The project/arrangement **version** and **variant** model (see
`07-project-versions-and-variants.md`) interacts with editing as follows:

- Different **project/arrangement versions** can record different media
  lineages and pipelines.
- **Track variants** can reference different clip pipelines and clip DSP chains.
- **Clip variants** can:
  - share the same base media asset but carry different ClipEditPipelines,
  - or reference different derived media.

This allows workflows such as:

- “Clean” vs “wildly processed” vocal variants,
- “Original comp” vs “tight edited” versions,
- alternate MIDI transform stacks for the same musical idea.

---

## 9. Performance Strategy

Pulse has the authority to decide:

- when to rely on **live evaluation** (cheap, low-latency operations),
- when to **ghost render** (audio and/or MIDI),
- when to **archive** and compress obsolete assets.

Signal is responsible for:

- executing Node-based DSP and ARA in realtime or online rendering contexts,
- providing progress/preview for heavy operations (render jobs),
- reporting performance characteristics for diagnostics.

The editing architecture ensures that all these decisions can be made without
compromising the core invariant:

> Original recorded/imported material is not silently lost; destructive-feeling
> workflows are implemented on top of a lineage-aware, nondestructive core.

---

## 10. Summary

This document defines the editing and nondestructive layer model for Loophole:

- **ClipEditPipelines** and **MidiEditPipelines** model stacked transforms,
  primarily reusing the existing **Node interface** where possible.
- **Clip-Safe Nodes** define a subset of Nodes allowed as offline/clip-level
  steps.
- **Media Editor** uses the same primitives but produces new **derived assets**
  rather than overwriting media in place.
- **Ghost renders** and **ghost MIDI** provide performance-friendly
  materialisations of complex pipelines.
- **ARA** and other external editors are integrated as pipeline steps in a
  defined position relative to clip DSP and track-level processing.
- **Archiving** compresses obsolete assets (e.g. to FLAC) while preserving
  lineage.
- **Undo, versions, and variants** are designed to coexist with this model
  without retrofitting.

This architecture ensures that editing in Loophole is powerful, reversible,
and scalable, while reusing as much of the Node ecosystem as possible for both
online and offline processing.
