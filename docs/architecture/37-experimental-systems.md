# Experimental Systems Architecture

This document defines **Experimental Systems** in Loophole — a formal framework
for designing, prototyping, testing, and evolving **non-standard, high-risk,
high-reward subsystems** that push the boundaries of what a DAW can do.

These systems:

- are *not* intended for v1,
- often explore unconventional DSP, UX, integration, or workflow ideas,
- may rely on speculative future hardware or emerging ML techniques,
- may eventually graduate into stable architecture documents,  
- or may remain permanently experimental.

The purpose of this document is **structural**: to define how experimental ideas
fit within the Loophole ecosystem and to ensure they never jeopardise core
determinism, safety, or stability.

It complements:

- `36-future-workflow-prototypes.md`
- `30-ai-assisted-audio-tools.md`
- `23-diagnostics-and-performance.md`
- `10-processing-cohorts-and-anticipative-rendering.md`
- `05-composer.md` & `35-composer-extended-intelligence.md`
- `02-signal.md` (engine)
- `03-pulse.md` (model)
- `04-aura.md` (UI)

Experimental systems share a common set of guarantees, boundaries, and
integration rules that prevent them from becoming architectural liabilities.

---

# 1. Purpose of Experimental Systems

The Experimental Systems layer exists to:

1. **Safely explore novel technologies**
   - Neural DSP  
   - Generative sequencing  
   - Adaptive UI surfaces  
   - Time-domain manipulation  
   - Spatial/VR/AR interfaces  
   - Cross-project intelligence

2. **Test radical ideas before committing to design changes**
   - Without entangling core subsystems,
   - Without increasing Pulse/Signal complexity prematurely.

3. **Provide a consistent “sandbox” environment**
   - where prototypes can be plugged in or removed cleanly.

4. **Allow user-facing “Labs” features**
   - Experimental toggleable modules,
   - Early-access features with gated functionality.

5. **Facilitate internal iteration**
   - Rapid prototyping with feedback loops,
   - Controlled exposure to selected users.

Experimental systems must **never** compromise:
- real-time safety,  
- project determinism,  
- IPC contract stability,  
- user data integrity.

---

# 2. Architectural Placement

Experimental systems sit **outside** the core architecture:

```
┌───────────────────────-┐
│         Aura           │
│  (UI & Experimental UI)│
└────────────┬───────────┘
             │
┌────────────▼──────────-─┐
│         Pulse           │
│  (Model, Rules, IPC)    │
└────────────┬────────-───┘
             │
┌────────────▼────────-───┐
│         Signal          │
│ (Audio Engine & DSP)    │
└─────────────────────────┘

 Experimental Systems
     ▲        ▲       ▲
     │        │       │
 Aura / Pulse / External Modules
```

Experimental Systems are **not a layer** but a *philosophical boundary*
governing how non-stable components attach to Aura or Pulse.

They can exist as:

- UI modules in Aura,
- high-level behavioural modules in Pulse,
- external “sidecar” processes that talk to Pulse,
- offline render modules,
- analysis services that Composer may call.

Signal should only expose deterministic, engine-safe hooks — experimental DSP
never runs in real-time unless explicitly promoted to stable status.

---

# 3. Categories of Experimental Systems

We classify experimental features into six groups:

1. **DSP Experiments**  
2. **Generative & Creative Experiments**  
3. **UI/Interaction Experiments**  
4. **Structural Data Model Experiments**  
5. **Composer/AI Experiments**  
6. **External Integration & Hybridisation Experiments**

Each category is described below.

---

# 4. Category 1 — DSP Experiments

DSP experiments involve novel processing techniques that are too early or too
unstable for the core engine.

Examples:

- Neural network–driven effects (neural amps, neural reverbs),
- Experimental spectral processors (phase vocoders, FFT reactors),
- Probabilistic plugin behaviours,
- Branch-predictive or adaptive DSP nodes.

### Architectural Requirements

DSP experiments must:

- run in **offline** or **sandbox** environments,
- communicate with Pulse through **batch processes** rather than continuous streams,
- optionally use the Render Pipeline,
- never run on the real-time thread until proven safe.

### Promotion Path

To graduate into the NodeGraph:

1. deterministic behaviour (or clearly non-deterministic marking),
2. predictable latency,
3. clear parameter semantics,
4. engine-safe thread model.

---

# 5. Category 2 — Generative & Creative Experiments

Includes any system that **creates** rather than **modifies**:

- generative MIDI engines,
- polyrhythmic pattern forgers,
- probabilistic drum machines,
- long-form “AI improvisers”.

These systems run **outside** core Clip/Lane editors and provide:

- suggestion layers,
- ghost clip previews,
- scene generation,
- brush-based generative tools.

### Interaction with Model

- Generate Clips/Lanes that go through normal Pulse editing.
- Must never directly mutate project state without user confirmation.
- May integrate with Clip Launcher as “generative pad” modes.

---

# 6. Category 3 — UI & Interaction Experiments

Experimental user interaction models include:

- VR/AR editing rooms,
- Pageless composition surfaces,
- Magnetic timeline surfaces,
- Force-directed node canvases,
- Hybrid sequencer–arranger views.

### Constraints

UI experiments:

- live entirely in Aura,
- use stable IPC,
- never alter Pulse’s conceptual model,
- may expose additional editor types.

This allows radical UI ideas without destabilising the DAW.

---

# 7. Category 4 — Structural Data Model Experiments

These experiments test deeper model evolution:

- multi-dimensional timelines (harmonic + rhythmic + spectral),
- non-linear clip networks,
- procedural arrangements,
- dynamic timeline objects.

### Safeguards

- Must wrap around **existing** Clips, Lanes, Tracks.
- Must rely on Pulse edit operations, not direct mutation.
- Must round-trip cleanly into standard project form.

---

# 8. Category 5 — Composer/AI Experiments

Examples:

- large-scale harmonic analysis,
- project fingerprinting for stylistic inference,
- long-form predictive arrangement engines,
- multi-agent “composer assistants” (e.g. arrangement agent, mix agent),
- experimental semantic embeddings.

### Requirements

- Live only inside Composer,
- Always optional and non-blocking,
- Exposed via Pulse as:
  - suggestions,
  - annotations,
  - optional transform previews.

These systems must remain advisory.

---

# 9. Category 6 — External Integration & Hybridisation

Future possibilities:

- modular hybrid environments (Max/MSP, Pure Data, Reaktor style),
- plugin-hosting inside AI tools,
- scriptable hybrid automation systems,
- integration with external knowledge bases,
- inter-app sync with non-music creative tools (video, VR storytelling).

### Architectural Requirements

- External modules communicate with Pulse via strict IPC.
- Pulse remains the sole state authority.
- Signal remains a real-time engine, untouched.

---

# 10. Experimental Lifecycle

Experimental systems follow a clear lifecycle:

## 10.1 Research → Prototype → Beta → Stable or Retire

### **Research**
- No code,
- Design exploration and feasibility.

### **Prototype**
- Implemented in isolation (Aura/sidecar process),
- No guarantees.

### **Beta**
- opt-in,
- documented limitations,
- uses stable IPC.

### **Stable**
- migrated into full architecture docs,
- added to main index.

### **Retired**
- archived in a “Retired Experiments” folder.

---

# 11. Constraints & Guarantees

Experimental systems:

- **must not** break undo/redo,
- **must not** introduce non-deterministic project data,
- **must not** require Pulse or Signal architectural overrides,
- **must** be IPC-contained,
- **must** be user-visible and optional,
- **must** run in isolation when unstable,
- **must** provide logging and diagnostics,
- **must** clearly indicate experimental nature.

---

# 12. Integration with Versioning & Collaboration

Future versions may support:

- experimental features enabled/disabled per user in collaboration,
- branching for experimental edits,
- editor modes that coexist with standard editing without conflict.

Pulse tracks experimental feature flags per session.

---

# 13. Experimental Sandbox Architecture

To support heavy experimentation, Loophole uses:

## 13.1 Plugin-Like Sandbox Containers

Experimental DSP modules run like sandboxed plugins:

- separate processes,
- crash-safe,
- optional GPU acceleration,
- optional model loaders.

## 13.2 Sidecar AI Engines

AI experiments run in dedicated processes:

- called asynchronously via Pulse,
- not wired into real-time systems.

## 13.3 Aura Prototype Workspace

A new area in Aura:

- toggled via “Experimental Mode”,
- isolates prototype UI views,
- exports debug overlays.

---

# 14. Future Possibilities

Experimental systems make Loophole extendable toward:

- VR/AR studio spaces,
- AI-driven co-producers,
- procedural arrangement engines,
- generative mix assistants,
- neural resynthesis instruments,
- multi-surface collaborative rooms,
- “logic nodes” for programming audio workflows.

All without ever needing a rewrite of Pulse or Signal.

---

# 15. Summary

The **Experimental Systems Architecture** provides a flexible,
future-proof environment for exploring advanced, unconventional,
and high-impact ideas without corrupting Loophole’s stable architecture.

It ensures:

- Signal stays real-time safe,
- Pulse stays authoritative and deterministic,
- Aura remains the flexible experiment host,
- Composer stays the intelligence and metadata layer,
- and IPC stays clean and extendable.

With this document, Loophole gains a long-term R&D runway — enabling innovation
without risking stability, identity, or performance of the core DAW.
