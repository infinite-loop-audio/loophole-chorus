# Comping Architecture

This document defines Loophole’s full **comping architecture** across audio,
MIDI, and automation domains.  
It establishes:

- take lanes  
- comp groups  
- swipe comping  
- comp layers  
- comp lanes vs clip lanes  
- nondestructive editing  
- integration with editing pipelines  
- integration with ARA  
- multi-track comp groups  
- comping in the Clip Launcher  
- rendering semantics  

Comping is a central part of modern recording workflows and must support
speed, flexibility and precision while preserving nondestructive guarantees.

This document builds on:

- `08-tracks-lanes-and-roles.md`  
- `09-clips.md`  
- `14-timebase-tempo-and-groove.md`  
- `15-advanced-clips.md`  
- `16-editing-and-nondestructive-layers.md`  
- `17-automation-and-modulation.md`  
- `18-midi-architecture.md`  
- `11-node-graph.md` (LaneStreams)

Pulse owns the comping model.  
Signal executes comp results in sample-domain playback and render modes.

---

## 1. Concepts

Loophole comping uses a **nondestructive layered model**:

- Recording produces **take lanes**, never overwriting existing audio.
- Swipe comping selects segments from takes into a **comp layer**.
- Comp layers are fully editable and reversible.
- A compiled **result clip** is created for playback.
- Comp layers and takes remain available for later revision.
- Multi-track comp groups allow phase-locked comping across tracks.

Automation and MIDI comping work analogously.

---

## 2. Take Lanes

Each Track maintains take lanes when comping is active.

```
TakeLane {
  laneId;
  clipId;          // audio or MIDI clip
  active: bool;
  muted: bool;
  metadata;
}
```

Properties:

- Take lanes are created automatically on punch-in, loop record, etc.
- Recording always produces a new take lane.
- Take lanes are ordered chronologically.
- Users can manually reorder, rename, merge or delete take lanes (never destructive).

### 2.1 Audio Take Structure

Each audio take lane contains a clip with:

- base audio media,
- warp map (optional),
- transient map,
- clip-lane settings (gain, fades, etc.).

Take lanes do **not** contain:

- ClipEditPipeline,
- ARA regions.

These only apply to the **result clip** or post-comp editing.

---

## 3. Comp Layers

A **Comp Layer** is a set of comp selections assembled from take lanes.

```
CompLayer {
  layerId;
  segments[];   // list of CompSegment
  muted: bool;
  metadata;
}
```

### 3.1 Comp Segment

```
CompSegment {
  sourceTakeLaneId;
  sourceClipRange;   // musical-time or sample-time
  resultRange;       // where it appears in the comp result
  crossfadeIn?;      // fade/crossfade boundaries
  crossfadeOut?;
}
```

Segments do not copy audio:  
they reference take-lane regions and are resolved during compilation.

### 3.2 Multiple Layers

- Multiple comp layers allow auditioning alternate comps.
- One active comp layer determines the **result clip**.
- Comp layers can be duplicated, merged, or branched.

---

## 4. Result Clips

The active comp layer produces a **result clip** in the Track’s main lane:

```
CompResultClip {
  clipId;
  compiledSegments[];
  ghostMedia?;         // optional compiled render
  metadata;
}
```

The result clip is what participates in:

- ClipEditPipeline,
- ARA,
- Clip DSP chain,
- Node graph,
- Clip Launcher,
- Rendering.

Pulse generates and maintains the result clip whenever comp edits change.

---

## 5. Audio Comping Mechanics

### 5.1 Swipe Comping

- User drags/swipes over take lanes.
- Pulse creates or updates CompSegments in the active CompLayer.
- Real-time auditioning switches between take lanes seamlessly.

### 5.2 Crossfades

- Automatic crossfades at segment boundaries.
- User-adjustable per-boundary fades.
- Phase-aligned fades for multi-track groups.

### 5.3 Clip Boundaries

- Comp boundaries align with musical time or transient markers.
- Pulse supports “snap to transient” behaviour for drum comping.

### 5.4 Warp & Timebase

Take-lane clips retain their own warp maps, but:

- Comp boundaries are resolved in musical time.
- Result clip’s warp map is derived from source maps.
- Optional “flatten warp” produces a straightened render.

---

## 6. ARA Integration

ARA (`15-advanced-clips.md`) only applies to the **compiled clip**, not takes.

### 6.1 ARA + Comp Lanes Interaction

When comping is modified:

- ARA regions become **out of sync**.
- Pulse either:
  - marks the ARA region invalid (most predictable), or
  - attempts partial re-sync if the comp change is small.

User workflow:

1. Build comp  
2. Apply ARA  
3. If comp changes → ARA regions flagged as “stale”  
4. User can re-render or reset ARA state

This prevents inconsistent editing across multiple takes.

---

## 7. Multi-Track Comp Groups

Multiple tracks may be linked into a **Comp Group**:

```
CompGroup {
  groupId;
  trackIds[];
  linkMode: "phaseLock" | "freeSync";
}
```

### 7.1 Phase-Locked Comping

- Edits on one track are mirrored on others.
- Transient and boundary alignment preserved.
- Crossfades computed in multi-track context.

### 7.2 FreeSync Comping

Tracks follow group boundaries but allow independent fine adjustments.

### 7.3 Multi-Track Editing

Operations supported:

- swipe comping across multiple tracks,
- phase-aligned splits and consolidations,
- multi-track crossfades,
- grouped comp layer switching.

---

## 8. MIDI Comping

MIDI clips also support comping via **MIDI Comp Layers**:

```
MidiCompLayer {
  layerId;
  segments[];  // ranges of notes/events
}
```

Segments reference:

- note ranges,
- CC ranges,
- expression segments.

MIDI comping use-cases:

- multiple takes of a solo,
- quantised vs unquantised expression,
- alternate improvisation variants.

Compilation produces a single **result MIDI lane**:

- merged effective notes,
- merged CC streams,
- MPE curves maintained or blended.

---

## 9. Automation Comping

Automation comping extends `17-automation-and-modulation.md`:

### 9.1 Automation Take Lanes

Each take can produce its own automation take:

```
AutomationTakeLane {
  laneId;
  points[];
}
```

### 9.2 Automation CompLayers

```
AutomationCompLayer {
  layerId;
  segments[];   // points or ranges
}
```

Compilation produces one automation envelope:

- per-parameter,
- per-lane or per-track,
- crossfaded or blended smoothly.

---

## 10. Editing Tools

### 10.1 Take Lane Tools

- swipe tool  
- pencil (insert comp boundary)  
- cut/split tools  
- range selection  
- audition tool (solo take lane)  
- reorder lanes  
- merge/re-balance takes  

### 10.2 Comp Layer Tools

- switch active comp layer  
- layer duplication  
- segment delete  
- crossfade editing  
- recompute boundaries  
- flatten comp to create a new take  

### 10.3 Result Clip Tools

- edit like any normal clip  
- apply ARA or ClipEditPipeline  
- bounce-in-place  
- consolidate  

---

## 11. Interaction with Editing Pipelines

After comping:

- The result clip’s audio becomes the input to the **ClipEditPipeline** (`16-editing-and-nondestructive-layers.md`).
- Edits in ClipEditPipeline do NOT affect take lanes.
- When comping changes:
  - ghost renders from clip edit pipelines are invalidated,
  - ARA regions are invalidated,
  - clip DSP chain remains intact.

Pulse ensures deterministic update ordering:

```
Take lanes → CompLayer → Result Clip → ClipEditPipeline → Clip DSP → Track Graph
```

---

## 12. Comping in Clip Launcher

Launcher clips may optionally contain:

- their own comp layers (rare), or
- directly reference the compiled clip from arrangement.

Launcher comp behaviour:

- clip loops the compiled region,
- per-take switching is disabled in Launcher,
- comp layers are frozen unless the user explicitly switches to “Launcher Editing Mode”.

This preserves stability during performance.

---

## 13. Rendering

During offline rendering:

Pulse resolves:

- active comp layer,
- compiled segments,
- crossfades,
- warp maps,
- clip pipelines (including ARA),
- automation comp layers,
- MIDI comp layers,
- timebase/tempo ramp alignment.

Signal receives a fully resolved sample-domain record of the comp result.

---

## 14. Undo & Versions

### 14.1 Undo

Undo stacks include:

- take-lane creation & deletion  
- comp segment edits  
- crossfade edits  
- comp layer operations  
- multi-track comp edits  

Take lanes and comp layers are always preserved for undo reconstruction.

### 14.2 Versions & Variants

Versions allow:

- alternate comp takes,
- alternate comping arrangements,
- alternate ARA breakdowns,
- clip variants based on different comp layers.

Variants allow:

- multiple comp interpretations of the same performance,
- quick A/B comparison,
- branching editing workflows.

---

## 15. Summary

This architecture defines a powerful, nondestructive comping system:

- take lanes for audio, MIDI and automation  
- comp layers with reversible editing  
- result clips feeding the full editing pipeline  
- ARA-aware invalidation and re-sync  
- multi-track comp groups  
- ghost render integration  
- Launcher compatibility  
- deterministic rendering  

Loophole’s comping system is designed to be fast, intuitive and robust, able to
handle simple takes or large multi-mic recording sessions with full phase
correctness and nondestructive lineage.
