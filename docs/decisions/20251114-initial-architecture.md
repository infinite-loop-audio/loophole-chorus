# Decision: Initial Multi-Process Architecture

- **ID:** 2025-11-14-initial-architecture  
- **Date:** 2025-11-14  
- **Status:** accepted  
- **Owner:** Infinite Loop Audio (Loophole core)  
- **Related docs:**  
  - `docs/architecture/01-overview.md`  

---

## 1. Context

Loophole is intended to be a next-generation Digital Audio Workstation that
addresses long-standing limitations in existing DAW design, including:

- Poor plugin organisation and taxonomy
- Monolithic architectures with tight coupling
- Limited UI flexibility, inhibiting experimentation
- Unreliable plugin hosting leading to crashes
- Difficulty iterating on UX without destabilising the engine
- Mixed real-time and non-real-time concerns within the same processes

Existing DAWs generally combine:

- UI rendering
- project state
- audio engine
- plugin hosting

…into a single application binary (sometimes multi-threaded, rarely multi-process).

This results in:

- heavy coupling
- complicated debugging
- fragile real-time invariants
- limited plugin sandboxing
- slow development cycles

Loophole aims to avoid these structural constraints by adopting a **modular,
multi-process architecture** from the beginning.

---

## 2. Problem Statement

We require an architecture that:

1. Prioritises **real-time safety**
2. Supports a **rich, expressive UI** unconstrained by RT constraints
3. Enables **independent iteration** of UI, model, and engine layers
4. Allows **plugin isolation** now or in the future
5. Provides **clear IPC contracts** between components
6. Scales to support:
   - future services
   - remote control surfaces
   - offline rendering processes
   - sophisticated plugin management workflows

The single-process model commonly used in DAWs is insufficient.

---

## 3. Options

### 3.1 Traditional Monolithic Application

**Description**

A single-process architecture where UI, project state, audio engine, and plugin hosting all run within one application binary.

**Pros**

- Simpler initial implementation
- Lower memory overhead
- Direct function calls between components

**Cons**

- Poor isolation: crashes in plugins or UI can take down the entire system
- Architectural rigidity: difficult to iterate on individual components
- Mixed real-time and non-real-time concerns
- Limited plugin sandboxing capabilities

**Conclusion**

Rejected due to poor isolation and architectural rigidity.

---

### 3.2 Multi-threaded but Single-Process

**Description**

A single-process architecture with multiple threads for UI, engine, and other concerns.

**Pros**

- Improved concurrency
- Some isolation through threading

**Cons**

- No crash isolation: a crash in any thread can still take down the entire process
- Does not improve iteration velocity for independent components
- Real-time constraints still affect all threads

**Conclusion**

Rejected. Improves concurrency but not crash isolation or iteration velocity.

---

### 3.3 "Everything in Rust"

**Description**

Implement the entire system in Rust, including UI, engine, and model layers.

**Pros**

- Strong type safety across the entire system
- No GC pauses
- Consistent language ecosystem

**Cons**

- Plugin hosting ecosystem support is immature
- UI frameworks are less developed than Electron/web technologies
- Real-time C++ DSP ecosystem is still superior for audio processing
- Would require significant ecosystem development

**Conclusion**

Rejected. Desirable in theory, but ecosystem maturity and migration path concerns make this impractical. Migration path remains open for a future decision.

---

### 3.4 UI-based DAW with Embedded Engine

**Description**

A primarily UI-driven architecture with the audio engine embedded within the UI process.

**Pros**

- Simpler process model
- Direct communication between UI and engine

**Cons**

- Weak real-time isolation
- UI crashes can take down the engine
- Engine constraints affect UI responsiveness

**Conclusion**

Rejected due to weak RT isolation.

---

### 3.5 Multi-Process Architecture

**Description**

A modular architecture with separate processes for UI (Aura), model (Pulse), engine (Signal), and meta (Chorus).

**Pros**

- Strong fault isolation
- Real-time guarantees for the engine
- UI freedom from RT constraints
- Clean state separation
- Explicit IPC contracts
- Extensibility for future services

**Cons**

- Increased system complexity
- Higher initial implementation cost
- Requires strict spec discipline
- IPC adds latency and overhead
- Multi-process debugging workflows required

**Conclusion**

Accepted. This architecture provides the best balance of isolation, flexibility, and long-term maintainability.

---

## 4. Decision

Loophole will adopt a **multi-process architecture** consisting of four major
repositories, each with its own responsibility boundaries:

---

### 4.1 Signal — Audio Engine (C++ / JUCE)

A dedicated **native** process responsible exclusively for **real-time audio and
plugin execution**.

Signal will:

- own audio/MIDI devices
- host plugin formats (VST3, CLAP, AU)
- maintain an internal audio graph
- apply parameter changes at sample accuracy
- process automation curves after they are resolved in Pulse
- provide telemetry frames to Aura
- expose a minimal IPC command surface

Signal will NOT:

- store project state
- manage undo/redo
- perform complex analysis on the RT thread
- resolve plugin taxonomy
- handle UI logic

---

### 4.2 Pulse — Data Model (TypeScript)

Pulse defines the core logical structure of a Loophole project.

Pulse will:

- maintain the source of truth for:
  - track structure
  - routing
  - plugin organisation
  - automation
  - clips/items/regions
  - metadata
- manage undo/redo and transactional updates
- derive engine graph instructions
- serialise project state
- validate changes before they reach Signal

Deployment model:

- Initially embedded within Aura
- Future decision may extract Pulse into its own service process

Pulse will NOT:

- perform audio processing
- execute plugins
- communicate directly with audio devices

---

### 4.3 Aura — User Interface (Electron / TypeScript)

Aura is the user-facing process that renders all visual layers of Loophole.

Aura will:

- render the arrange, mixer, editors, workspace
- interact with Pulse for project logic
- relay engine commands to Signal
- display telemetry
- host plugin UIs in native windows
- provide creative and organisational tooling for plugins

Aura will NOT:

- directly manipulate engine-internal structures
- maintain its own source of truth
- bypass Pulse for project changes
- perform any real-time audio work

---

### 4.4 Chorus — Meta Repository

Chorus owns the **architectural truth** of the entire Loophole ecosystem.

Chorus will contain:

- architecture documentation
- IPC schemas
- data model specifications
- real-time guidelines
- decision documents
- the Meta-Protocol
- agent rules for Cursor/Codex
- tasks for distributed development

Chorus will NOT contain:

- runtime code
- binaries
- implementation artefacts

Chorus is the governing reference for all other repositories.

---

## 5. Rationale

This architecture was chosen because it provides:

### 5.1 Strong Fault Isolation

A crash in a plugin or the UI cannot take down the engine.

### 5.2 Real-Time Guarantee

Signal can be engineered to strict RT constraints without UI interference.

### 5.3 UI Freedom

Aura can use any modern web-based technologies without impacting engine
stability or plugin correctness.

### 5.4 Clean State Separation

Pulse can evolve independently with advanced organisational features.

### 5.5 Explicit Contracts

IPC schemas and meta-tools formalise all cross-layer communication.

### 5.6 Extensibility

Future sub-processes (sandboxed plugins, render nodes, headless servers) can be
added without restructuring.

---

## 6. Consequences

### 6.1 Positive

- Independent development velocity per repo
- Easier debugging (issues are isolated to specific processes)
- Safer plugin hosting
- Clear IPC mechanisms facilitate distributed architecture
- System becomes more maintainable over time
- Documentation-first discipline improves quality

### 6.2 Negative / Trade-offs

- Increased system complexity
- Higher initial implementation cost
- Requires strict spec discipline (Chorus is critical)
- IPC adds latency and overhead (must be carefully managed)
- Development requires multi-process debugging workflows
- Running multiple processes consumes slightly more memory
- Requires robust CI validation across repos
- Developers must learn and trust the Meta-Protocol

---

## 7. Follow-Up Actions

1. Define IPC message structures in `docs/specs/ipc/`
2. Create decision document for Pulse extraction into separate service process
3. Define plugin sandboxing options
4. Specify telemetry streaming format
5. Plan render nodes architecture
6. Design cross-process plugin UI mechanisms

---

## 8. Notes

- This decision forms the foundation of all subsequent architecture decisions.
- Future decisions will build upon this multi-process foundation.

---
