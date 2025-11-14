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
- **Pulse** — project/model state (complex NRT logic)
- **Aura** — user interface (expressive, unconstrained)
- **Chorus** — coordination, contracts, specifications

## 1.2 Multi-Process Model

Where possible, major responsibilities are isolated into separate processes:

- Prevent plugin crashes from taking down the UI
- Allow independent scaling and optimisation
- Allow sophisticated visual UI frameworks without compromising engine safety

## 1.3 Declarative Contracts

Communication between layers uses explicit schemas defined in `@chorus:/docs/specs/`.
Each layer operates using declared interfaces, not assumptions.

## 1.4 Replaceability

Any layer can be replaced independently:

- UI themes or frameworks
- rewritten engine
- standalone Pulse service
- remote control surfaces
- headless renderers

This modularity is central to Loophole’s long-term evolution.

---

# 2. Normative Language

This document uses **normative language** consistent with IETF standards.
These keywords indicate requirements and expectations that apply to all Loophole
implementations and specifications:

- **MUST / MUST NOT**
  - A strict requirement of the architecture.
  - Violations will lead to undefined behaviour or incompatibilities.

- **SHOULD / SHOULD NOT**
  - A recommended practice that should be followed unless a justified reason exists.

- **MAY**
  - An optional capability or behaviour.

Whenever such terms appear in this document or others in the **Chorus**
repository, they are meant in the formal sense defined above.

---

# 3. Repository Overview

The Loophole ecosystem consists of four repositories:

| Repo | Role | Language | Notes |
|------|------|----------|-------|
| **Signal** | Real-time engine | C++ (JUCE) | Plugin hosting, audio graph, RT state |
| **Pulse**  | Model layer      | TypeScript | Project, routing, automation, taxonomy |
| **Aura**   | UI layer         | Electron + TS | Views, editors, workspace, telemetry |
| **Chorus** | Meta repository  | Markdown + JSON | Specs, ADRs, IPC, meta-protocol |

Chorus anchors the architecture; the runtime repos implement it.

---

# 4. Runtime Components

Loophole’s runtime consists of:

- **Signal process** — native audio engine
- **Aura process** — UI + embedded Pulse
- **(Future)** Pulse process — isolated model layer
- IPC channels between processes defined in `@chorus:/docs/specs/`

---

# 5. Layer Responsibilities

Each layer’s responsibilities are defined normatively.

## 5.1 Signal (Audio Engine)

**Primary Responsibilities:**

- Manage audio/MIDI devices
- Maintain the audio processing graph
- Host plugins (VST3, AU, CLAP)
- Guarantee real-time behaviour
- Provide telemetry
- Apply sample-accurate parameter changes
- Expose an IPC command surface

**Signal MUST NOT:**

- Store project structures
- Perform non-real-time computations
- Handle UI logic
- Block or allocate in RT code

---

## 5.2 Pulse (Project and Data Model)

Pulse is the authoritative project-state owner.

**Primary Responsibilities:**

- Tracks, routing, groups
- Plugin models and taxonomy
- Automation data
- Clip/region/item structures
- Undo/redo
- Project serialisation
- Deriving RT-safe graph instructions for Signal

Initially Pulse is embedded in Aura; later it MAY run as its own service.

---

## 5.3 Aura (User Interface Layer)

Aura provides all interaction and visual elements.

**Primary Responsibilities:**

- Arrange/editor/mixer views
- Plugin UIs (native windows)
- Engine control (via Signal IPC)
- Project manipulation (via Pulse)
- Displaying engine telemetry
- Managing plugin browsing and categorisation

Aura MUST NOT perform audio processing.

---

## 5.4 Chorus (Meta Repository)

Chorus defines:

- architectural documents
- specifications
- ADRs
- IPC schemas
- meta-protocol
- agent rules

Chorus MUST contain no runtime code and serves as the system’s source of truth.

---

# 6. Inter-Process Communication (IPC)

IPC contracts between processes live under:

```
@chorus:/docs/specs/
```

IPC includes:

- Command messages
- Telemetry streams
- Parameter updates
- Graph-change instructions
- Lifecycle messages (initialisation, shutdown)

No implicit behaviour is permitted; all expectations MUST be documented.

---

# 7. Real-Time Safety

Any operation that might violate RT constraints MUST NOT occur in the audio
callback. This includes:

- memory allocation
- locking
- I/O
- logging
- container resizing

Pulse MUST provide RT-safe immutable graph instructions that Signal can swap
atomically.

---

# 8. Evolution Strategy

The architecture is designed for long-term extensibility:

- Pulse MAY be extracted to its own process
- Plugin sandboxing MAY be added
- Telemetry MAY expand to richer analysis formats
- Remote/UI extensions MAY communicate over network IPC

Such changes require new ADRs and spec updates.

---

# 9. Document Conventions

- British English
- Stable heading structure
- All file references use `@repo:/path/` shorthand
- Internal examples use triple-backtick fences
- AI-generated files MUST be wrapped in quadruple backticks

---

# 10. Summary

This document defines the conceptual structure of the Loophole system.

Chorus defines the architecture.
Pulse defines the project state.
Signal defines the audio engine.
Aura defines the interface.

All runtime behaviour MUST conform to the responsibilities defined here.

---
