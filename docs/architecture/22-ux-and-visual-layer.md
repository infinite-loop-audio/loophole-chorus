# UX & Visual Layer Architecture

This document defines the **user experience and visual architecture** for
Loophole, as implemented primarily in **Aura**.

It describes:

- the overall UX philosophy,
- the view hierarchy and core workspaces,
- navigation and layout principles,
- editing paradigms in the arrangement, launcher and editors,
- media browser and library presentation at a UX level,
- discoverability and guidance,
- theming, visual density and readability,
- responsiveness to different screen setups,
- interaction with windowing & plugin UI architecture (`24-windowing-and-plugin-ui.md`),
- integration with control surfaces (`21-control-surfaces-and-assistive-hardware.md`).

This is not a pixel-perfect design spec, but a **structural UX architecture**:
it must be stable enough to influence Aura’s code and layout decisions and to
keep later visual design iterations coherent.

It builds on:

- `04-aura.md`
- `06-processing-cohorts-and-anticipative-rendering.md`
- `08-tracks-lanes-and-roles.md`
- `09-clips.md`
- `10-parameters.md`
- `13-media-architecture.md`
- `14-timebase-tempo-and-groove.md`
- `15-advanced-clips.md`
- `16-editing-and-nondestructive-layers.md`
- `17-automation-and-modulation.md`
- `18-midi-architecture.md`
- `19-comping-architecture.md`
- `20-rendering-and-offline-processing.md`
- `21-control-surfaces-and-assistive-hardware.md`
- `24-windowing-and-plugin-ui.md`

---

## 1. UX Philosophy

Loophole’s UX is driven by a few core principles:

1. **Immediate musicality**  
   - You should be able to make sound within seconds of opening the app.
   - Hardware should work out of the box.
   - Core actions (record, play, create track, add instrument) must be always
     within easy, obvious reach.

2. **Depth without clutter**  
   - The surface stays clean and readable by default.
   - Complexity is revealed progressively as the user digs deeper.
   - Power features are highly available but not visually noisy.

3. **One mental model, many views**  
   - Arranger, Launcher, Mixer, Editors and Browser all reflect a single
     underlying project model.
   - Moving between views should never feel like “switching apps”.

4. **Context is king**  
   - The UI adapts to what you’re doing:
     - if you’re editing clips, controls focus on clip editing;
     - if you’re mixing, controls focus on channels, sends and VCA;
     - if you’re performing, Launcher dominates.
   - Control surfaces mirror that context.

5. **Non-destructive and reversible**  
   - The UI encourages safe experimentation.
   - Editing layers, comping, ghost renders, automation and versions all
     surface clearly as “states”, not hidden behaviours.

6. **Accessible & legible**  
   - Good contrast, scalable UI, readable typography.
   - No reliance on tiny text or target hitboxes.
   - Keyboard and control-surface access wherever possible.

---

## 2. Core Workspaces

Aura provides several **primary workspaces**:

1. **Project Home / Overview**
2. **Arrangement View**
3. **Clip Launcher View**
4. **Editor Views**
   - Piano Roll
   - Audio Editor
   - Expression/Automation Editor
   - MIDI Drum Sequencer
5. **Mixer View**
6. **Media & Library View**
7. **Diagnostics & Performance View** (see `25-diagnostics-and-performance.md` when defined)
8. **Settings & Profiles**

All workspaces share:

- a common **header bar** (transport, tempo, project info),
- a common **side dock** for Browser/Inspector,
- consistent theming and interaction patterns.

### 2.1 Multi-View Composition

On a typical desktop:

- Arrangement + Mixer + Browser may coexist:
  - Arrangement is centre stage,
  - Mixer docked bottom or right,
  - Browser on left or right.

On smaller screens:

- Views become **tabbed** or accessible via a quick switcher.
- Launcher overlays Arrangement rather than coexisting side-by-side.

---

## 3. View Hierarchy

At a high level:

```
RootShell
 ├── GlobalHeader (transport, tempo, status, project)
 ├── MainArea
 │    ├── PrimaryView (Arrangement | Launcher | Mixer | Editor | Library)
 │    └── SecondaryView(s) (dockable Mixer, Editor, Browser)
 └── GlobalFooter (optional: status, CPU/memory, hardware status)
```

Aura’s layout system should support:

- persistent layout presets (per-machine),
- keyboard-driven view toggling,
- smooth transitions when views change (no jarring reflows).

---

## 4. Navigation Model

Key navigation principles:

1. **Global hotkeys** for:
   - play/stop/record,
   - toggle Arrangement / Launcher / Mixer,
   - open Browser,
   - open active Clip Editor,
   - focus next/previous Track,
   - jump to markers/scenes.

2. **Command palette**:
   - searchable command interface,
   - command categories (Project, Track, Clip, Mixer, View, Hardware),
   - integration with control-surface learn mode.

3. **Context-sensitive right-click menus**:
   - always show relevant actions,
   - avoid deep nesting.

4. **Breadcrumb & context indicators**:
   - obvious indicators of:
     - current track,
     - current clip,
     - current editor focus,
     - current hardware context (if relevant).

---

## 5. Arrangement View UX

Arrangement View is the canonical “timeline”:

- tracks vertically,
- time horizontally,
- clips as primary visual elements.

Key UX features:

1. **Track Header Area**
   - track name, colour, icon,
   - arming/monitor/solo/mute,
   - visibility toggles for lanes (MIDI, audio, video, automation),
   - quick access to Node Graph summary (node badges).

2. **Track Body**
   - hybrid lanes model:
     - instrumentation / NodeGraph summary at top,
     - clip lanes beneath,
     - automation lanes optional below or overlaid.
   - clear differentiation between:
     - clip content,
     - automation envelopes,
     - comping layers (if visible).

3. **Hybrid Track UX**
   - Single “Track” concept can host:
     - instrument content,
     - multiple audio lanes,
     - video lanes,
     - automation,
     - nested child tracks.
   - UI provides simple default (single lane, single instrument),
     but seamlessly reveals additional lanes and structures as needed.

4. **Clips & Lanes**
   - clip borders and content clearly convey:
     - lane membership,
     - comping state,
     - ARA presence,
     - clip edit pipelines (icon or badge),
     - loop state, stretch state.

5. **Smooth Editing**
   - minimal mode switching:
     - trim, move, split via clear handles and gestures,
     - clip duplication with modifier keys,
     - consolidated vs raw take lanes toggled via a single control.

6. **Launcher Overlay**
   - Launcher can slide over Arrangement (see `20-clip-launcher.md`),
     while vertical track alignment is preserved.
   - Alignment grid and clip handles allow drag-and-drop between
     Arrangement and Launcher.

---

## 6. Clip Launcher UX

Clip Launcher View is a **matrix** aligned to tracks:

- rows = tracks,
- columns = scenes / clip slots.

Key UX behaviours:

1. **Shared Track Identity**
   - Launcher rows share names, colours, icons with Arrangement tracks.
   - Muting/soloing/arming behaviour is unified across views.

2. **Scenes**
   - scenes column at the left,
   - global scene launch controls,
   - per-scene follow actions (in future),
   - per-scene “playthrough” function:
     - plays each scene in order for the length of the longest clip, with
       shorter clips looping for the scene duration.

3. **Interaction**
   - click to trigger / stop clips,
   - keyboard shortcuts for scenes/clips,
   - hardware grid controller mirrors Launcher layout.

4. **Arrangement Integration**
   - context menu actions:
     - “Capture Scene to Arrangement”,
     - “Create Scene from Time Selection”,
     - “Copy Clip to Launcher / Arrangement”.
   - drag-and-drop between views.

---

## 7. Editor Views

Editor views are focused spaces for detailed manipulation.

### 7.1 Piano Roll

Key features:

- note lanes with scale highlighting,
- “folded” view to show only active pitches,
- ghost notes from other clips/tracks (optional),
- velocity and per-note expression lanes,
- tools for quantise, groove, transform,
- clear linking to MIDI pipelines (e.g. quantise is a transform, not a one-way smash).

### 7.2 Audio Editor

Key features:

- waveform view with:
  - transient markers,
  - warp markers,
  - clip edit pipelines overlay (badges & markers),
  - ARA presence (if active).

- Comping overlays (when editing a comp result):
  - ability to see source segments,
  - quick jump to take lanes.

- Non-destructive vs “destructive-feeling” editing:
  - UI labels operations clearly:
    - “Commit edit (creates new media version)”
    - “Apply clip-only transform”
  - history clearly shows media lineage.

### 7.3 Automation / Expression Editor

Shared automation editor across:

- clip automation,
- track automation,
- per-note & per-parameter expression.

Features:

- vector curves, tension controls,
- drawing tools,
- snapping and quantise,
- lane stacking and grouping,
- safe overview of multiple parameters without clutter.

### 7.4 MIDI Drum Sequencer

The MIDI Drum Sequencer provides a grid-based view optimised for percussion
programming and loop-based composition.

Key features:

- a fixed or user-defined list of drum lanes (Kick, Snare, Hi-Hat, Perc, etc.),
- per-lane note assignment and remapping using track-level or instrument-level definitions,
- step-sequencer-style grid with:
  - step resolution options,
  - velocity and probability lanes,
  - per-step variations (flam, roll, ratchet),
  - per-lane articulation controls,
- loop controls and pattern-length tools independent of clip length,
- preview-on-hover and audition for rapid workflow,
- integration with groove and quantisation pipelines,
- optional "pattern bank" sidebar for storing drum variations,
- tight integration with pad/grid controllers (Launchpad, Push, etc.), with live LED feedback.

The Drum Sequencer shares the same core MIDI editing model as the Piano Roll
but offers a specialised workflow for percussive and pattern-driven music.

### Additional Future Editors

Loophole may introduce further specialised editors (e.g., Sampler Editor,
Pattern/Chord Editor, Device Editor, Video Clip Editor) as the ecosystem
grows. Each editor will define its own interaction rules while remaining
consistent with the overall UX architecture laid out in this document.

---

## 8. Media & Library UX (High-Level)

Detailed media system is covered in `13-media-architecture.md`.  
This section focuses on **UX aspects only**.

Key elements:

1. **Browser Pane**
   - dockable to left or right,
   - contains tabs:
     - Samples & Loops,
     - Instruments & Effects,
     - Presets & Devices,
     - Project Assets (recordings, renders, bounces),
     - Favourites & Collections.

2. **Search & Filter**
   - global search bar (by name, tag, type),
   - filters:
     - length/duration,
     - key/scale,
     - tempo,
     - timbre/texture tags,
     - instrument/role tags.

3. **Contextual Recommendations**
   - integration with Composer:
     - “Suggested for this track”,
     - “Similar to current clip”,
     - “Matches project tempo/key”.

4. **Audition UX**
   - preview synced to project tempo,
   - ability to audition through selected track’s processing chain,
   - one-click drag into:
     - Arrangement,
     - Launcher,
     - Drum/instrument lanes.

5. **Visualisation**
   - waveforms,
   - mini spectrograms for audio,
   - MIDI snippet previews,
   - tag badges.

The detailed “XO-like” cloud or more experimental browser modes can be specified in a dedicated media-browser doc later; this doc just ensures the Browser is a first-class citizen in the main layout.

---

## 9. Discoverability & Guidance

Loophole must avoid the “black box” feeling of many DAWs.

Guidance systems:

1. **First-Run Experience**
   - guided flow:
     - choose audio interface,
     - create first track (instrument or audio),
     - plug in a keyboard/controller and confirm it works,
     - record something quickly.

2. **Inline Hints**
   - small, unobtrusive hints for:
     - drag-and-drop targets,
     - what clicking certain regions will do,
     - options when hovering over timeline/track header.

3. **Command Palette & Search**
   - all features discoverable by search,
   - command entries have descriptions and (where relevant) keyboard shortcuts.

4. **Context Help**
   - pressing e.g. `F1` shows:
     - context-sensitive short doc or quick help,
     - link to more detailed docs.

---

## 10. Theming, Density & Readability

Visual design guidelines:

- high-contrast mode and dark mode baseline,
- typography tuned for long-session use,
- minimal use of heavy borders; rely on spacing, shading and grouping,
- avoid pixel-perfect tiny affordances; use larger hit areas,
- adjustable density:
  - “Comfortable” vs “Compact” modes,
  - Mixer can be compact while Arrangement remains more spacious.

Theming engine must support:

- semantic colours (status, track types, warnings, etc.),
- track colour as an accent throughout,
- plugin-specific accenting in plugin host views.

---

## 11. Windowing & Multiscreen Behaviour

Windowing specifics are in `24-windowing-and-plugin-ui.md`.  
This doc emphasises a few UX requirements:

1. **Layout Persistence per Screen Set**
   - arrangement of:
     - main window,
     - mixers,
     - editors,
     - plugin UIs,
   must persist per machine + screen set.

2. **Safe Recovery**
   - no windows may become unreachable when screens are unplugged:
     - minimum size constraints,
     - auto-recentre if window is fully off-screen.

3. **Workspace Snapshots**
   - users can save named workspace layouts, including:
     - which views are open,
     - how they are docked,
     - how control surfaces are bound.

---

## 12. Control Surface UX Integration

Control surface behaviour (as per `21-control-surfaces-and-assistive-hardware.md`) must be reflected clearly in the UI:

1. **Indicator of Active Hardware Context**
   - a small HUD or status indicator showing:
     - which device is controlling what,
     - which bank/page is active,
     - which context (Mixer/Arrangement/Plugin/Launcher) is bound.

2. **Mapping Visualiser**
   - an inspector that:
     - highlights parameters as you move hardware controls,
     - shows mapping connections in real time.

3. **Learn Mode UX**
   - simple workflow:
     - click “Learn”,
     - click GUI parameter,
     - move control,
     - mapping is created,
     - preview mapping and confirm.

4. **Conflict Warnings**
   - gently warn when a control is mapped to conflicting things in the same context,
   - propose resolutions (layered mapping, context restriction, etc.).

---

## 13. Performance & Diagnostics UX (Hooks)

The diagnostics system is detailed in its own document.  
From a UX perspective:

- CPU/memory graphs:
  - optional, non-intrusive, accessible on demand.
- Per-track and per-node performance hints:
  - simple badges (“heavy”, “clipped”, “denoised by anticipative engine”).
- Clear display when:
  - anticipative rendering is buffering,
  - processing cohorts are rebalancing,
  - ghost renders are being updated or re-rendered.

---

## 14. Accessibility

Key accessibility requirements:

- scalable UI (font size and zoom),
- keyboard-driven operation for all major workflows,
- screen-reader-considerate structure (for future accessibility work),
- colour-blind-friendly defaults and palettes.

Aura should plan for and not preclude future deeper accessibility support.

---

## 15. Summary

This document defines the UX and visual layer architecture for Loophole:

- coherent workspaces (Arrangement, Launcher, Mixer, Editors, Browser),
- consistent navigation and context signalling,
- a powerful yet unobtrusive media browser,
- strong editing ergonomics (clips, comping, pipelines, automation),
- deep integration with Launcher and advanced playback/recording models,
- persistent and safe windowing/multiscreen behaviour,
- clear integration with control surfaces,
- a strong foundation for theming, density and accessibility.

The intent is that future visual design passes and implementation details fit
within this structure, rather than fighting against it.
