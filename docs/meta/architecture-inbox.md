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
