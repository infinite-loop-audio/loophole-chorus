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

### Node Determinism Classification
**Tag:** Composer / Signal / Pulse  
**Priority:** P1  
**Status:** incorporated  
Composer should collect behavioural telemetry to determine whether a plugin
is deterministic or non-deterministic. This allows Pulse to safely place nodes
into the anticipative cohort and Signal to avoid unpredictable behaviour in the
background engine.
