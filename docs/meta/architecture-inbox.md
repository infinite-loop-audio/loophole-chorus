# Architecture Inbox

This document collects ideas, observations, discussions, and speculative features
that arise during Loophole’s design process. Items recorded here are not yet
commitments. They exist to preserve insight without blocking ongoing work or
disrupting architectural flow.

Entries in this inbox may be refined, merged, escalated into formal
architecture documents, incorporated into IPC specifications, or discarded.

---

## Format for New Entries

Each new idea should follow this structure:

```
## <Short Title>
**Tag:** <Engine / Pulse / Aura / IPC / Workflow / DSP / UX / Composer / Misc>  
**Priority:** P1 | P2 | P3  
**Status:** proposed | accepted | incorporated | rejected  
A concise description of the idea, why it matters, and when it should be
considered. Include cross-references if relevant.
```

---

## Entries

### Processor Determinism Classification
**Tag:** Composer / Signal / Pulse  
**Priority:** P1  
**Status:** incorporated  
Composer should collect behavioural telemetry to determine whether a plugin
is deterministic or non-deterministic. This allows Pulse to safely place processors
into the anticipative cohort and Signal to avoid unpredictable behaviour in the
background engine.

### Dynamic Cohort Switching UX Indicators
**Tag:** Aura / UX  
**Priority:** P3  
**Status:** proposed  
Aura may display a small indicator showing whether a track or processor is running
in the live or anticipative domain. Useful for debugging performance and improving
user intuition around engine behaviour.

### High-Rate Gesture Stream
**Tag:** IPC / Pulse / Signal  
**Priority:** P2  
**Status:** proposed  
Add a dedicated IPC side-channel for ultra-fast parameter gestures (e.g., fast knob
movements), to avoid JSON overhead during continuous control. Pulse should still
validate and route gestures but not block real-time flow.

### Processor Identity Aliasing
**Tag:** Pulse / Composer  
**Priority:** P2  
**Status:** proposed  
Support alias definitions so projects remain stable if a plugin changes identity
after an update. Composer may help suggest identity mappings across versions
and formats.

### Track-Level Freeze Visualisation
**Tag:** Aura / UX / Signal  
**Priority:** P3  
**Status:** proposed  
Visually represent when anticipative rendering is effectively “freezing” a track so
users understand the balance between live and pre-rendered DSP.

### Multi-Clip Lane Expansion
**Tag:** Pulse / Aura  
**Priority:** P3  
**Status:** proposed  
Allow clips to define new Lanes that are not part of the Track's static layout,
enabling per-clip experimentation with audio or MIDI layering.

### SQLite-Based Storage Architecture
**Tag:** Composer / Misc  
**Priority:** P2  
**Status:** proposed  
Use SQLite (or similar SQL database) for storage of project data and settings/config that don't require optimised data streams (e.g., excluding raw audio samples). Requires a general database for Composer sync and settings, plus investigation into whether project files should be or contain SQLite databases. Need to evaluate constraints and limitations of this approach.

### Gesture-Based Tool Selection
**Tag:** Aura / UX  
**Priority:** P3  
**Status:** proposed  
Enable gesture-based tool selection in editors (e.g., piano roll) by holding an action key and performing mouse gestures to switch modes. Maintain keyboard shortcut fallbacks (e.g., number keys) for traditional workflows. Improves workflow fluidity for users who prefer gesture-based interaction.

### Context-Aware Plugin Suggestions
**Tag:** Composer / Aura / UX  
**Priority:** P2  
**Status:** proposed  
Implement processing contexts that detect track characteristics (e.g., drums, vocals, bass) and prioritize relevant plugin categories when loading FX. For example, a drum track would prioritize Drum category plugins, followed by Compressors, Shapers, etc. Reduces cognitive load and speeds up workflow by surfacing contextually relevant processors first.

### Visual Plugin Browser
**Tag:** Composer / Aura / UX  
**Priority:** P2  
**Status:** proposed  
Transform plugin/processor browsing into a visual experience with interface thumbnail captures, intelligently-picked conceptual colors (matching the plugin's "feel" rather than static collection colors), thematic icons from a wide selection (potentially auto-applied), routing diagrams (e.g., multi-out configurations), determinism indicators, and plugin stability metrics. Enables faster visual recognition and selection of processors.

### Fine-Grained Track Nesting Controls
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Design extensive UI elements for fine-grained control of track nesting and folder organization. Existing DAW implementations make it awkward to move tracks between folders, into nested structures, or adjust nesting levels. Aura should provide intuitive drag-and-drop, keyboard shortcuts, and visual feedback for reorganizing track hierarchies without disrupting workflow.

### Extensible Action and Shortcut Configuration
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Implement fully extensible and customizable keyboard, action, and macro shortcut configuration system. Support mapping any control signature (mouse, keyboard, MIDI controller, etc.) to any action with simple conflict resolution. Allow multiple gestures to map to a single action, with gestures contextual to the active window. Enable in-place action discovery with optional hover tooltips and easy in-place assignment of shortcuts. Empowers users to create personalized workflows and adapt the interface to their preferred input methods.

### Predictable UI Layout and Window Management
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Design Aura's UI layout to be visually similar to other DAWs but with a much more predictable interaction model. Main arrangement windows contain the full sequence, with an optional clip launcher interface able to replace or partially cover it (similar to Studio One 7). Clip/lane editors, mixer, media library, etc. should have easy shortcuts by default and replace the whole internal window space for focused editing work. A secondary level of editor window can be attached in-place where appropriate for smaller editing facilities, similar to Bitwig/Ableton's bottom strip. This layered approach provides both full-screen focus and contextual secondary tools without disrupting the primary workflow.

