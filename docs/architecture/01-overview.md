# Loophole Architecture Overview

This document provides a complete architectural overview of the **Loophole**
Digital Audio Workstation ecosystem. It defines the conceptual responsibilities
of each subsystem, describes runtime boundaries, and provides a holistic view of
the multi-process model.

This document is the primary human-readable introduction to the Loophole system.

---

# 1. Architectural Principles

Loophole is designed around the following principles:

## 1.1 Clear Separation of Concerns

Real-time (RT) and non-real-time (NRT) responsibilities must never mix.
This protects audio stability, prevents UI stalls from affecting audio, and
enables a more flexible system architecture.

- **Signal** — real-time safe, minimal, deterministic
- **Pulse** — model layer (complex operations allowed; off RT thread)
- **Aura** — UI (unbounded complexity permitted)
- **Chorus** — architectural system of record

## 1.2 Multi-Process Model

Where possible, major responsibilities are isolated into separate processes:

- Prevent plugin crashes from taking down the UI
- Allow independent scaling and optimisation
- Allow sophisticated visual UI frameworks without RT penalties

## 1.3 Declarative Contracts

Communication between layers uses explicit schemas defined in `@chorus:/docs/specs/`.
Each layer operates using *declared interfaces*, not implicit assumptions.

## 1.4 Replaceability

Any layer can be replaced independently:

- A new UI style/theme
- A rewritten engine
- A separate Pulse service
- A headless renderer
- Experimental data modelling

This modularity is key for long-term evolution.

---

# 2. Repository Overview

The Loophole ecosystem consists of four repositories:

| Repo | Role | Language | Notes |
|------|------|----------|-------|
| **Signal** | Real-time engine | C++ (JUCE) | Plugin hosting, audio graph, RT state |
| **Pulse**  | Model layer | TypeScript | Project, routing, automation, taxonomy |
| **Aura**   | Interface layer | Electron + TS | Views, editors, workspace, telemetry |
| **Chorus** | Meta-repo | Markdown + JSON | Specs, ADRs, IPC, docs, meta-protocol |

Chorus anchors the architecture.
The other three repos implement it.

---

# 3. Runtime Components

Loophole’s runtime consists of:

- **Signal process** — native application hosting the audio engine
- **Aura process** — the main UI + embedded Pulse
- **(Future)** Pulse as a separate process for improved isolation
- IPC channels between processes defined in `@chorus:/docs/specs/`

---

# 4. Layer Responsibilities

Each layer’s responsibilities are defined *normatively* (i.e., as binding rules).

## 4.1 Signal (Audio Engine)

**Primary Responsibilities:**

- Own and manage audio/MIDI devices
- Maintain the audio processing graph
- Host plugin formats (VST3, AU, CLAP)
- Guarantee real-time behaviour
- Apply parameter changes sample-accurately
- Stream telemetry (meters, transport, analysers)
- Expose an IPC surface for:
  - Graph instructions
  - Parameter changes
  - Transport commands
  - Telemetry subscriptions

**Signal DOES NOT:**

- Store project structures
- Manage high-level plugin organisation
- Own undo/redo
- Perform complex/lazy computations on the audio thread
- Render plugin UI

This strict boundary keeps the engine minimal and robust.

---

## 4.2 Pulse (Project and Data Model)

Pulse is the **source of truth** for Loophole’s logical project state.

**Primary Responsibilities:**

- Project structure:
  - Tracks, buses, groups
  - Clips, regions, items
  - Timeline, tempo map, markers
- Routing model:
  - Sends, sidechains, track I/O
- Plugin models:
  - Instances
  - Metadata
  - Categorisation/taxonomy
  - Parameter schemas
- Automation representation
- Undo/redo engine
- Serialization / persistence
- Deriving engine-ready graph instructions for Signal

**Initial Deployment Model:**

- Pulse is embedded inside the Aura process.

**Future Model (ADR TBD):**

- Extract Pulse into its own service to improve isolation.

---

## 4.3 Aura (User Interface Layer)

Aura provides the user experience layer.
It is free to be expressive and computationally heavy.

**Primary Responsibilities:**

- Arrange, edit, mix, and manage content
- Render plugin/editor UIs (native or generic)
- Provide plugin discovery and categorisation UI
- Relay user operations to Pulse (state changes)
- Relay engine control to Signal (transport, commands)
- Display engine telemetry
- Handle keyboard, pointer, and gesture events
- Save/load projects via Pulse

**Aura DOES NOT:**

- Perform any audio processing
- Hold authoritative project state
- Maintain undo/redo independently
- Bypass Pulse to communicate directly with Signal (except where specified by IPC rules)

Aura is intentionally “fat” — designed for creativity, responsiveness, and discoverability.

---

## 4.4 Chorus (Meta Repository)

Chorus provides the governing framework for:

- Architectural documents (`docs/architecture/`)
- Specifications (`docs/specs/`)
- Decisions (`docs/decisions/`)
- AI meta-protocol (`docs/meta/`)
- Inter-repo tasks (`tasks/`)
- Cursor rules (`.cursor/rules/`)
- Conventions and guidelines

Chorus is authoritative.
Other repos MUST conform to Chorus.

---

# 5. Inter-Process Communication (IPC)

IPC contracts between processes live under:

```
@chorus:/docs/specs/
```

IPC includes:

- Command messages
- Telemetry streams
- Parameter updates
- Plugin editor window requests
- Lifecycle messages (initialisation, shutdown, heartbeat)

No assumptions between layers are permitted without being documented here.

---

# 6. Real-Time Safety

Any operation that might violate RT constraints MUST NOT occur in Signal’s audio
processing callback.

This includes:

- Memory allocation
- I/O
- Locks
- Maps, vectors, or containers that may resize
- Plugin scanning
- Logging
- Dynamic string manipulation
- Any non-O(1) behaviour

Pulse must derive real-time-safe, immutable graph instructions for Signal.

---

# 7. Evolution Strategy

The architecture is designed to evolve:

- Pulse may become a separate process
- Telemetry protocols may expand
- Plugin sandboxes may be added
- Aura may support multiple windows or remote control surfaces
- Render nodes may be introduced (offline rendering)

Such changes require:

1. A new ADR
2. Updated specs
3. Updated architecture docs
4. Coordinated updates across Signal, Pulse, and Aura

---

# 8. Document Conventions

- British English
- Stable, predictable section headings
- Cross-references must use `@repo:/path` notation
- Internal code blocks use fenced triple backticks
- Entire documents generated by AI must be wrapped in quadruple backticks
- Maximum clarity, minimum ambiguity

---

# 9. Summary

This document establishes the structural foundation for all Loophole components.

Chorus defines the architecture.
Pulse defines the project state.
Signal defines the audio engine.
Aura defines the interface.

All runtime behaviour MUST conform to the responsibilities defined here.

---
