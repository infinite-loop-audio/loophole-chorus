# Editor Architecture

This document defines the **unified editing platform** for all Loophole editors:
Piano Roll, Drum Sequencer, Audio Editor, Automation/Expression Editor, Clip
Editor, Video Editor, ARA Editor, and any future specialised editors.

It provides:

- the core editor state model,
- tools and editing primitives,
- selection semantics,
- timeline/grid behaviour,
- transaction + undo/redo model,
- gesture integration,
- media-type–specific behaviours,
- IPC responsibilities,
- and forward-compatible extension points.

It complements:

- `09-clips.md`
- `15-advanced-clips.md`
- `16-editing-and-nondestructive-layers.md`
- `17-automation-and-modulation.md`
- `18-midi-architecture.md`
- `19-comping-architecture.md`
- `20-rendering-and-offline-processing.md`
- `21-control-surfaces-and-assistive-hardware.md`
- `26-ux-and-visual-layer.md`

---

# 1. Goals

Loophole’s editors must:

1. **Share a unified conceptual model**  
   No editor feels “foreign”; all follow the same underlying rules.

2. **Be expressive, powerful, discoverable**  
   Tools should feel natural with mouse, tablet, pen, touch or hardware.

3. **Be non-destructive by default**  
   Destructive actions route through editing pipelines.

4. **Integrate fully with gesture and value-stream workflows**  
   From pointer drags to high-resolution hardware encoders.

5. **Provide robust undo/redo**  
   Grouped by user intent, not raw primitives.

6. **Work on arbitrary media types**  
   MIDI, audio, automation, ARA regions, video, etc.

7. **Scale to future editors**  
   Sampler, spectral, chord, pattern, spatial, VR/AR editors.

8. **Use Pulse as the single source of truth**  
   Editors exist entirely conceptually in Aura; Pulse owns data.

---

# 2. Editor Categories

Primary editors:

1. **Piano Roll** – MIDI notes & per-note expression  
2. **Drum Sequencer** – step grid & pattern logic  
3. **Audio Editor** – slicing, warping, transient editing, pipelines  
4. **Automation / Expression Editor** – parameter curve editing  
5. **Clip Editor** – multi-lane content orchestration (master editor)

Extension editors:

- **ARA Editor** – pitch/timing/phoneme editing per clip  
- **Video Editor** – frame/time alignment, transform, opacity, rotation  
- **Future Editors** – Sampler, Device, Spectral, Pattern, etc.

All inherit from the **Editor Core**.

---

# 3. Editor Core Model

```
Editor {
  editorId;
  type: pianoRoll | drumSequencer | audio | automation | clip | ara | video | custom;
  target: ClipId | TrackId | NodeId | LaneId;
  viewState: EditorViewState;
  selection: EditorSelection;
  tool: ActiveTool;
  snapping: SnappingSettings;
  focus: FocusState;
  config: EditorConfig;
}
```

### 3.1 View State

```
EditorViewState {
  zoom;
  scroll;
  timeRange;
  laneRange?;
  visualOverlays[];
}
```

### 3.2 Selection Model

```
EditorSelection {
  items: SelectionItem[];
  anchor?;
  range?;
  mode: replace | add | remove | toggle;
}
```

Selection items always reference canonical Pulse IDs.

---

# 4. Editing Primitives

Primitives are the atomic edit instructions sent to Pulse:

### 4.1 Time
- move, slip  
- trim_start / trim_end  
- stretch  
- split / join  
- warp / align  

### 4.2 Value
- set_value  
- scale_value  
- curve_value  
- interpolate  

### 4.3 Structure
- add_note / remove_note  
- add_slice / remove_slice  
- lane_add / lane_delete  
- pipeline_step_add  

### 4.4 Transform
- quantise  
- nudge  
- fold/unfold  
- velocity brush (MIDI)  
- pitch brush (ARA)  

### 4.5 Batch
- consolidate  
- bounce_clip  
- commit_pipeline  

Everything in every editor reduces to one of these primitives or a batch thereof.

---

# 5. Tools

Tools are UI constructs in Aura that emit primitives.

### 5.1 Universal Tools
- Pointer  
- Blade (split)  
- Pencil (draw notes, points, steps)  
- Eraser  
- Range tool  
- Transform tool  
- Time-stretch tool  

### 5.2 Editor-Specific Tools
- **Piano Roll:** articulation tool, velocity brush  
- **Drum Sequencer:** probability brush, step toggle  
- **Automation:** curve tool, tension brush  
- **Audio:** slice tool, warp marker, transient align  
- **ARA:** pitch line tool, phoneme tool  
- **Video:** transform/crop/opacity  

Tools do not manipulate data directly; they generate primitives.

---

# 6. Selection System

### 6.1 Selection Items

```
SelectionItem {
  type: note | step | automationPoint | audioSlice | clip | region | lane | timeRange;
  id;
}
```

### 6.2 Selection Behaviour
- click → replace  
- shift-click → extend  
- cmd/ctrl-click → toggle  
- drag → box selection  
- alt-drag → time-range selection  

Selection is type-aware and validated by Pulse.

---

# 7. Snapping & Grid

All editors use a unified grid model:

```
Grid {
  resolution;
  adaptive: bool;
  swing?;
  followGroove?;
}
```

Snapping modes:
- to grid  
- to nearest  
- magnetic  
- disabled  

Grid derives from track, global tempo, or user preference.

---

# 8. Editor Transactions (Undo/Redo)

### 8.1 Transaction Lifecycle

```
BeginTransaction(label)
 → EditPrimitive*
EndTransaction
```

Triggers:
- gesture start/end,
- tool actions,
- keyboard shortcuts,
- script execution.

### 8.2 Nested Transactions
- quantise, consolidate, comping  
- Pulse supports nested rollback if sub-operations fail.

---

# 9. Editor Lifecycle & Focus

### 9.1 Lifetime
Aura creates editors; Pulse validates targets.

### 9.2 Focus
Focus determines:
- shortcut routing  
- hardware mapping context  
- gesture ownership  
- selection ownership  

Aura ensures only one editor is “hot” at a time.

---

# 10. Gesture Integration

See `gesture-and-streaming.md`.

Editors integrate directly with:

- pointer gestures  
- stylus/pen  
- multi-touch  
- controller faders/encoders  
- XY pads  
- footswitches  

### 10.1 Gesture Types
- draw gesture  
- time-move gesture  
- value gesture  
- scrub gesture  
- range gesture  

Gesture → high-res stream → Pulse decimation → primitives → commit.

---

# 11. Editor-Specific Behaviour

---

## 11.1 Piano Roll

- notes with pitch, duration, velocity, articulation  
- multi-lane per-note expression  
- fold by scale/chord  
- ghost notes from other clips  
- transform ops: transpose, scale, stretch  

---

## 11.2 Drum Sequencer

- per-voice lanes  
- step toggles  
- probability, velocity, repeats  
- independent pattern length  
- audition on hover  

---

## 11.3 Audio Editor

- transient detection overlays  
- warp markers  
- slicing  
- slip & stretch  
- destructive pipelines  
- time/pitch operations  
- ARA-enabled deep editing  

---

## 11.4 Automation & Expression

- vector curves  
- multi-parameter view  
- tension & shape editing  
- interpolation tools  
- envelope grouping  
- per-note expression integration  

---

## 11.5 Clip Editor (master)

- multi-lane layout  
- lane add/remove  
- lane routing (LaneStream assignment)  
- lane mixing controls  
- hybrid media per clip  
- drag content between clips  

---

## 11.6 ARA Editor

- phoneme segmentation  
- pitch line editing  
- timing grids  
- formant tools  
- analysis region mapping  
- integration with clip pipelines  

---

## 11.7 Video Editor

- frame preview  
- slip/stretch  
- transform handles  
- opacity/rotation automation  
- timeline sync  

---

# 12. IPC Integration

### Aura → Pulse
- `editor.create`
- `editor.setViewState`
- `editor.setSelection`
- `editor.setTool`
- `editor.beginTransaction`
- `editor.endTransaction`
- `editor.edit`
- `editor.gesture.start/update/end`

### Pulse → Aura
- `editor.updateSelection`
- `editor.updateViewState`
- `editor.transactionConfirmed`
- `editor.previewGenerated`

### Pulse ↔ Signal
- preview waveforms  
- transient maps  
- warp previews  
- clip pipeline previews  

---

# 13. UX Expectations

Editors must:

- remain fluid even during heavy DSP  
- never block when media is offline/downloading  
- show visual hints for handles, grids, selections  
- unify tool interaction across editor types  
- auto-scroll and auto-zoom where intuitive  
- fully integrate control surfaces  

From `26-ux-and-visual-layer.md`.

---

# 14. Future Extensions

Supported without structural change:

- spectral editors  
- sampler keygroup editors  
- device editors  
- chord/pattern editors  
- spatial 3D automation editors  
- VR/AR editing  
- hybrid timeline editors  

All via new primitives, tools, selection types.

---

# 15. Summary

The editor architecture provides:

- a unified model for selection, tools, snapping and editing,
- robust gestures and transactions,
- consistent behaviour across all media types,
- clear IPC boundaries between Aura, Pulse and Signal,
- flexible extension points for future editors.

This architecture ensures that Loophole’s editing experience feels coherent,
powerful and modern across MIDI, audio, automation, ARA, video and beyond.
