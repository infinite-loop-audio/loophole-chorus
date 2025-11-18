# Clip Launcher Architecture

This document defines the architecture for Loophole’s **Clip Launcher**, a
nonlinear performance and prototyping interface aligned directly with the
arrangement tracks and timeline.

The Clip Launcher supplements — but does not replace — the linear arrangement.
It enables loop-based composition, rapid auditioning of variations, scene
sequencing and performance-driven workflows, without introducing a separate
“mode”. The Launcher slides over the arrangement in Aura and shares the same
Track structure.

This document covers the conceptual model and Pulse responsibilities. The IPC
domain (Pulse <→ Aura and Pulse <→ Signal) will be specified separately.

---

## 1. Goals

The Clip Launcher must:

1. Provide a **Track-aligned** nonlinear clip grid (no separate mode).
2. Allow **free movement** of Clips between arrangement and launcher.
3. Structure launcher playback into **Scenes**, each a vertical slice of
   Slots across Tracks.
4. Allow **looping launcher clips** by default with per-Slot launch rules.
5. Support **quantised clip/scene launching** synced to the project timebase.
6. Integrate smoothly with linear arrangement playback (override or hybrid).
7. Support **Scene Playthrough**: sequentially audition all Scenes as a full
   arrangement prototype.
8. Enable population of launcher Slots from:
   - existing Clips,
   - Pad Workspace performances,
   - Kit Builder outputs,
   - imported media.
9. Remain conceptually simple: the launcher is an **overlay** on existing
   Tracks/Lanes, not a parallel track system.
10. Provide clear IPC semantics between Aura ⇄ Pulse ⇄ Signal.

---

## 2. Core Concepts

### 2.1 Scene

A **Scene** is a column in the Clip Launcher, identified by:

- `sceneId` — stable identifier,
- `name`,
- `colour`,
- `index` (ordering left-to-right).

A Scene groups Slots across Tracks. Triggering a Scene launches all occupied
Slots in that column according to quantisation rules.

Scenes do not overlap or occupy temporal positions in the linear arrangement.

Scenes are **not** Sections, but optionally map to them for convenience.

---

### 2.2 Launcher Slot

A **Launcher Slot** represents optional Clip content at the intersection of a
Track and a Scene:

- implicit identifier `(trackId, sceneId)`
- optional `clipRef`:
  - reference to an existing Clip in a Lane, or
  - reference to a “LauncherClip” stored in a launcher-only pool,
    structurally identical to Clip but not placed on the timeline.
- launch parameters:
  - `quantise` — bar, beat, 1/2, 1/4, or global default,
  - `mode` — loop / play-once / play-then-return-to-arrangement,
  - `gain`, `pan`, and other per-Slot transform overrides (optional future
    detail),
- status (Pulse → Aura):
  - `idle`, `queued`, `playing`, `stopping`, `stopped`,
  - current loop index when playing.

Slots **do not duplicate audio content**. They only reference Clips or
Clip-like structures.

---

### 2.3 Launcher State

Pulse maintains global LauncherState:

- `activeSceneId` (or `null`)
- `activeMode`:
  - `arrangement-only`,
  - `launcher-only`,
  - `hybrid` (launcher overrides active Track content)
- per-Track activation state:
  - which Slot is playing for each Track,
  - when transitions are scheduled to occur (quantised positions)
- Scene Playthrough state:
  - current scene in playthrough sequence,
  - scheduling / progress,
  - loop counters per Slot.

---

### 2.4 Launcher vs Arrangement Audio Precedence

Default behaviour:

- If a Slot is playing on Track X:
  - That Track’s arrangement content is **muted** until the Slot stops.

Hybrid variations may be added later, but v1 assumes:

- **Launcher overrides arrangement** on a per-Track basis.

Automation, timebase, mixer state, and global playback all continue normally.

---

## 3. Playback & Quantisation Model

Launcher playback uses the **same Timebase** as arrangement playback.

### 3.1 Quantised Launch

Launch operations (Scene or Slot):

- never occur immediately unless quantise = “off”,
- normally schedule for the next quantisation boundary (bar/beat/etc.),
- may require per-Slot overrides or global quantise settings.

Signal uses the same quantisation grid for actual playback start times.

---

### 3.2 Slot Playback Modes

Slots support:

- `loop`  
  Default. Clip loops indefinitely while Scene is active.

- `play-once`  
  Clip starts at quantised boundary, plays once, stops.

- `play-then-return`  
  Clip plays once, then hands control back to the arrangement for that Track.

Additional modes (gate, retrigger, one-shot-with-tail) can be added over time.

---

### 3.3 Scene Playback

Launching a Scene:

1. Pulse validates all Slots belonging to that Scene.
2. Pulse schedules launches for all occupied Slots at the next quantised time.
3. Pulse updates LauncherState accordingly.
4. Pulse requests Signal to start playback for each Slot’s Clip.

Stopping a Scene:

- Stops all Slots from that Scene,
- or if in `hybrid` mode, hands Tracks back to arrangement playback.

---

### 3.4 Scene Playthrough

Scene Playthrough is a **non-linear audition mode** that plays Scenes in
sequence, each for a duration equal to:

- the **longest Clip** in that Scene’s occupied Slots,
- with shorter Clips looping until the Scene’s time elapses.

Playthrough operation:

1. User triggers “Playthrough”.
2. Pulse sets `activeSceneId` to the first Scene.
3. Slots in Scene A launch and play for the computed Scene duration.
4. At Scene boundary:
   - Pulse transitions to Scene B with quantised scheduling.
   - Short-looping Slots are stopped/restarted as needed.
5. Continue until final Scene.
6. On escape/stop:
   - All Slots stop,
   - arrangement resumes based on `activeMode` policy.

Playthrough provides fast “preview the track structure” functionality without
committing Clips to the arrangement.

---

## 4. Interaction with Tracks, Lanes and Arrangement

### 4.1 Track Alignment

The Launcher aligns directly with Track Y-axis order.

- Every Track may have a LauncherRow.
- Multi-Lane Tracks still appear as single rows in the Launcher for simplicity,
  but each Slot’s ClipRef determines which Lane executes the audio.

### 4.2 Slot Clip Assignment

Users can:

- drag arrangement Clips into Slots:
  - results in a reference to that Clip’s data, not a copy,
- drag Slots into arrangement:
  - results in the Clip being instantiated as a timeline Clip,
- copy Clips both ways,
- create new launcher-only Clips that can later be committed to Lanes.

This supports rapid prototyping before committing to the full arrangement.

### 4.3 Arrangement-Launcher Visual Integration (Aura)

Launcher slides over arrangement in Aura. This provides:

- a single, unified vertical track stack,
- direct drag-and-drop between views,
- no “toggle mode” friction.

---

## 5. Scenes, Sections and Project Metadata

Scenes and Sections serve different conceptual roles:

- **Sections**: semantic regions of the arrangement timeline.
- **Scenes**: vertical collections of loop triggers.

However, usability improves by mapping them loosely:

- Scenes may optionally be labelled with Section names.
- Aura may auto-suggest mapping based on:
  - Clip content,
  - arrangement positions,
  - user workflow habits.

Later, Composer may propose Scene layouts based on Section metadata.

---

## 6. Integration with Kits and Pad Workspaces

The Launcher synergises strongly with Kits and Pad Workspaces.

### 6.1 Kit → Scene Population

A Kit can populate a Scene automatically:

- Kit slots become Launcher Slots,
- each populated with an auto-generated or user-selected Clip,
- ideal for drum-based scene building.

### 6.2 Pad Workspace → Slot/Scene

Pad performances can be committed to Slots:

- “Record pad performance as Scene N’s Slots”
- “Commit selection of pads to specific Track Slots”
- “Bounce pad pattern into an arrangement Clip”

### 6.3 Scenes as Nonlinear Prototyping Space

Workflow example:

1. Build a drum kit via Kit Builder.
2. Jam on Pad Workspace.
3. Commit pad patterns into Scene 1.
4. Create variations in Scenes 2, 3…
5. Playthrough Scenes to audition full structure.
6. Drag Scenes or individual Slots into arrangement to produce the final track.

This hybrid supports both loop-based and linear composition styles, and allows
fast ideation without committing choices prematurely.

---

## 7. Pulse-Level Model

### 7.1 Data Objects

```
Scene {
    sceneId: string;
    name: string;
    colour: optional;
    index: number;     // ordering
}

LauncherSlot {
    trackId: string;
    sceneId: string;
    clipRef?: ClipRef;
    quantise?: QuantiseValue;
    mode?: SlotMode;
}

LauncherState {
    activeSceneId?: string;
    activeMode: 'arrangement' | 'launcher' | 'hybrid';
    trackStates: Map<trackId, SlotPlaybackState>;
    playthroughState?: {
        sequence: sceneId[];
        currentIndex: number;
        remainingBars: number;
    };
}
```

These objects live entirely in Pulse. Aura receives snapshots and events. Signal
only receives actual playback instructions derived from this state.

---

### 7.2 Pulse <→ Signal Responsibilities

Pulse is responsible for:

- preparing launch schedules,
- quantising start/stop requests,
- determining loop boundaries for playthrough,
- resolving ClipRefs into audio regions,
- sending explicit playback instructions to Signal:
  - “play Clip X on Track Y at quantised time Z”,
  - “stop playback for Track Y at time Z”.

Signal is responsible for:

- performing the actual audio playback of referenced media,
- handling looping and retriggering according to Pulse timing,
- reporting any runtime errors (missing media, decode failures).

Pulse **never** assumes media availability; it queries the Media domain before
issuing playback commands.

---

## 8. IPC Overview (Pulse <→ Aura)

Commands (Aura → Pulse):

- `launcher.addScene`, `launcher.removeScene`, `launcher.renameScene`,
  `launcher.reorderScenes`
- `launcher.setSlotContent`
- `launcher.clearSlot`
- `launcher.setSlotQuantise`
- `launcher.setSlotMode`
- `launcher.startScene`
- `launcher.stopScene`
- `launcher.startSlot`
- `launcher.stopSlot`
- `launcher.setActiveMode`
- `launcher.playthrough.start(sequence?)`
- `launcher.playthrough.stop`

Events (Pulse → Aura):

- `launcher.sceneAdded/Updated/Removed/Reordered`
- `launcher.slotUpdated/Cleared`
- `launcher.sceneStateChanged`
- `launcher.slotStateChanged` (playing/stopped, loop index)
- `launcher.playthroughStateChanged`

IPC details live in the domain spec; this document provides the architecture.

---

## 9. Relationship to Other Domains

- **Tracks, Lanes and Clips (06, 07)**  
  LauncherSlots reference Clips; Clips remain the canonical event sequence.
  The launcher never replaces or mutates arrangement Clips.

- **Nodes (09)**  
  Launcher-triggered playback uses the same Channel and Node graph as
  arrangement playback.

- **Timebase (Timebase IPC Domain)**  
  All launcher operations depend on consistent quantised scheduling.

- **Automation (08)**  
  Arrangement automation continues running while launcher content plays.
  Launcher playback does not generate automation events.

- **Media (12)**  
  Launcher Slots referencing missing media still exist; they simply fail to
  launch and visually indicate unavailability.

- **Kits and Pad Workspaces (12)**  
  The launcher provides a home for kit-based and pad-based prototyping.

---

## 10. Summary

The Clip Launcher provides a dynamic, fluid, nonlinear layer on top of Loophole’s
existing linear sequencing architecture.

Key qualities:

- aligned directly with Tracks,
- sliding overlay for fast drag-and-drop between launcher and arrangement,
- Scenes structure nonlinear performance,
- Slots reference Clips rather than duplicating media,
- quantised launch system using Timebase,
- Scene Playthrough enables rapid pre-arrangement preview,
- strong integration with Kits and Pad Workspaces,
- clean Pulse model that delegates playback timing to Signal,
- Aura UI can explore expressive layouts without burdening engine complexity.

This architecture allows the Clip Launcher to be a powerful creative engine
without disrupting the deterministic, model-centric core of Loophole.
