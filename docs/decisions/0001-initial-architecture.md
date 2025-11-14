# ADR 0001 — Initial Multi-Process Architecture

## Status
**Accepted**

## Date
2025-11-14

---

# 1. Context

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

# 2. Problem Statement

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

# 3. Decision

Loophole will adopt a **multi-process architecture** consisting of four major
repositories, each with its own responsibility boundaries:

---

## 3.1 Signal — Audio Engine (C++ / JUCE)

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

## 3.2 Pulse — Data Model (TypeScript)

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
- Future ADR may extract Pulse into its own service process

Pulse will NOT:

- perform audio processing
- execute plugins
- communicate directly with audio devices

---

## 3.3 Aura — User Interface (Electron / TypeScript)

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

## 3.4 Chorus — Meta Repository

Chorus owns the **architectural truth** of the entire Loophole ecosystem.

Chorus will contain:

- architecture documentation
- IPC schemas
- data model specifications
- real-time guidelines
- ADRs
- the Meta-Protocol
- agent rules for Cursor/Codex
- tasks for distributed development

Chorus will NOT contain:

- runtime code
- binaries
- implementation artefacts

Chorus is the governing reference for all other repositories.

---

# 4. Rationale

This architecture was chosen because it provides:

### 4.1 Strong Fault Isolation
A crash in a plugin or the UI cannot take down the engine.

### 4.2 Real-Time Guarantee
Signal can be engineered to strict RT constraints without UI interference.

### 4.3 UI Freedom
Aura can use any modern web-based technologies without impacting engine
stability or plugin correctness.

### 4.4 Clean State Separation
Pulse can evolve independently with advanced organisational features.

### 4.5 Explicit Contracts
IPC schemas and meta-tools formalise all cross-layer communication.

### 4.6 Extensibility
Future sub-processes (sandboxed plugins, render nodes, headless servers) can be
added without restructuring.

---

# 5. Consequences

## 5.1 Positive

- Independent development velocity per repo
- Easier debugging (issues are isolated to specific processes)
- Safer plugin hosting
- Clear IPC mechanisms facilitate distributed architecture
- System becomes more maintainable over time
- Documentation-first discipline improves quality

## 5.2 Negative

- Increased system complexity
- Higher initial implementation cost
- Requires strict spec discipline (Chorus is critical)
- IPC adds latency and overhead (must be carefully managed)
- Development requires multi-process debugging workflows

## 5.3 Neutral / Trade-offs

- Running multiple processes consumes slightly more memory
- Requires robust CI validation across repos
- Developers must learn and trust the Meta-Protocol

---

# 6. Alternatives Considered

### 6.1 Traditional Monolithic Application
Rejected due to poor isolation and architectural rigidity.

### 6.2 Multi-threaded but single-process
Improves concurrency but not crash isolation or iteration velocity.

### 6.3 “Everything in Rust”
Desirable in theory, but:
- Plugin hosting ecosystem support is immature
- UI frameworks are less developed
- Real-time C++ DSP ecosystem is still superior
- Migration path remains open for a future ADR

### 6.4 UI-based DAW with embedded engine
Rejected due to weak RT isolation.

---

# 7. Notes

- Future ADRs will define:
  - IPC message structures
  - Pulse extraction
  - Plugin sandboxing options
  - Telemetry streaming format
  - Render nodes
  - Cross-process plugin UIs

- This ADR forms the foundation of all subsequent architecture decisions.

---

# 8. Decision Made
Loophole adopts a **modular, multi-process architecture** with four core
repositories and strict separation of real-time, model, UI, and meta concerns.

---
