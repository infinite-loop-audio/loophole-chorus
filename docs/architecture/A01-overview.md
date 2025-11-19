# Loophole Architecture Overview
This document provides a complete architectural overview of the **Loophole** Digital Audio Workstation ecosystem. It defines the conceptual responsibilities of each subsystem, describes runtime boundaries, and provides a holistic view of the multi-process model. This document is the primary human-readable introduction to the Loophole system.

---

# 1. Architectural Principles

Loophole is designed around the following principles:

## 1.1 Clear Separation of Concerns

Real-time (RT) and non-real-time (NRT) responsibilities must never mix. This protects audio stability, prevents UI stalls from affecting audio, and enables a more flexible system architecture.

- **Signal** — real-time safe, minimal, deterministic
- **Pulse** — project/model state (complex NRT logic)
- **Aura** — user interface (expressive, unconstrained)
- **Composer** — shared knowledge metadata service
- **Chorus** — coordination, contracts, specifications

## 1.2 Multi-Process Model

Where possible, major responsibilities are isolated into separate processes:

- Prevent plugin crashes from taking down the UI
- Allow independent scaling and optimisation
- Allow sophisticated visual UI frameworks without compromising engine safety

## 1.3 Declarative Contracts

Communication between layers uses explicit schemas defined in `@chorus:/docs/specs/`. Each layer operates using declared interfaces, not assumptions.

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

This document uses **normative language** consistent with IETF standards. These keywords indicate requirements and expectations that apply to all Loophole implementations and specifications:

- **MUST / MUST NOT**
  - A strict requirement of the architecture.
  - Violations will lead to undefined behaviour or incompatibilities.

- **SHOULD / SHOULD NOT**
  - A recommended practice that should be followed unless a justified reason exists.

- **MAY**
  - An optional capability or behaviour.

Whenever such terms appear in this document or others in the **Chorus** repository, they are meant in the formal sense defined above.

---

# 3. Repository Overview

The Loophole ecosystem consists of five repositories:

| Repo        | Role                          | Language         | Notes                                           |
|------------|-------------------------------|------------------|------------------------------------------------|
| **Signal** | Real-time engine              | C++ (JUCE)       | Plugin hosting, audio graph, RT state          |
| **Pulse**  | Model layer                   | Rust             | Project, routing, automation, taxonomy         |
| **Aura**   | UI layer                      | Electron + TS    | Views, editors, workspace, telemetry           |
| **Composer** | Knowledge metadata service  | TBD              | Plugin/parameter metadata, roles, mappings     |
| **Chorus** | Meta repository               | Markdown + JSON  | Specs, ADRs, IPC, meta-protocol                |

Chorus anchors the architecture; the runtime repos implement it. Composer enriches runtime behaviour with metadata but MUST NOT be required for project integrity.

---

# 4. Runtime Components

Loophole's runtime consists of:

- **Signal process** — native audio engine
- **Pulse process** — standalone server process implemented in Rust, owning the authoritative project model, routing topology and IPC orchestration between Aura and Signal
- **Aura process** — UI layer
- **Composer service** — external shared metadata service (HTTP, optional)
- IPC channels between processes defined in `@chorus:/docs/specs/`

Composer runs out-of-process and is accessed via web-style APIs by Pulse. If Composer is unavailable, Loophole MUST continue to function using local metadata and project data.

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
- Execute dual-engine DSP architecture with separate schedulers for live and anticipative processing cohorts

**Signal MUST NOT:**

- Store project structures
- Perform non-real-time computations
- Handle UI logic
- Block or allocate in RT code

---

## 5.2 Pulse (Project and Data Model)

Pulse is the authoritative project-state owner. Pulse is implemented as a **separate server process** in **Rust from the outset**, not as a TypeScript layer within Aura. The earlier idea of Pulse as a TypeScript layer embedded in Aura has been superseded by the decision to run Pulse as its own Rust service from the beginning.

**Primary Responsibilities:**

- Tracks, routing, groups
- Plugin models and taxonomy
- Automation data
- Clip/region/item structures
- Undo/redo
- Project serialisation
- Deriving RT-safe graph instructions for Signal
- Integrating Composer metadata (where available)
- Maintaining parameter aliasing and remapping tables
- Assigning nodes to processing cohorts (live vs anticipative) based on liveness requirements, deterministic behaviour metadata, and dependency closure

Pulse communicates with Aura and Signal via IPC, owning the long-lived, authoritative data model.

Pulse MAY query Composer for:

- plugin and parameter metadata,
- semantic roles,
- mapping suggestions across plugin versions and formats,
- plugin categorisation and tags.

Pulse MUST treat Composer data as advisory and MUST ensure that project state remains valid without Composer.

---

## 5.3 Aura (User Interface Layer)

Aura provides all interaction and visual elements.

**Primary Responsibilities:**

- Arrange/editor/mixer views
- Plugin UIs (native windows)
- Engine control (via Signal IPC)
- Project manipulation (via Pulse)
- Displaying engine telemetry
- Managing plugin browsing and categorisation (using Pulse/Composer metadata)

Aura MUST NOT perform audio processing.

Aura MUST NOT talk directly to Signal or Composer; all runtime communication MUST go through Pulse.

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

## 5.5 Composer (Knowledge Metadata Service)

Composer is an external service providing shared plugin and parameter metadata.

**Primary Responsibilities:**

- Normalise plugin identity across formats and versions
- Describe parameters (names, types, ranges, scaling, groups)
- Assign semantic roles to parameters (e.g. `filter.cutoff`, `fx.mix`)
- Suggest mappings between plugin versions and formats
- Suggest mappings between different plugins based on roles
- Aggregate anonymised, opt-in user signals for naming, tagging and mapping
- Provide deterministic-behaviour metadata that indicates whether nodes are safe for anticipative rendering or must remain in the live cohort

**Composer MUST NOT:**

- Be required for core project loading or playback
- Store or receive audio, Clip contents or detailed arrangement data
- Communicate directly with Signal or Aura

Composer MAY be unavailable; in that case Pulse MUST fall back to local metadata and heuristics. Composer's deterministic-behaviour metadata contributes to Pulse's cohort assignment decisions, enabling safe anticipative rendering of deterministic nodes while keeping non-deterministic or live-dependent nodes in the live cohort.

---

## 5.6 Processing Cohorts and Anticipative Rendering

Loophole employs a dual-engine DSP architecture that separates processing into two processing cohorts (PC): **live** and **anticipative**. This design enables extremely low-latency interaction for live performance whilst simultaneously performing heavy DSP on large buffers in the background, allowing Loophole to scale to projects that would otherwise exceed real-time CPU limits.

The **live cohort** handles nodes that require sample-accurate response to user input, live audio/MIDI input, or contain non-deterministic DSP. These nodes run at the audio callback rate with short buffers (64–128 samples) and always respond instantly to gestures.

The **anticipative cohort** processes deterministic nodes ahead of the playhead using very large buffers (hundreds of milliseconds to several seconds). Signal maintains a render horizon that stays ahead of the playhead, effectively pre-rendering entire sections of the project in the background.

Pulse owns all cohort assignment decisions, determining membership based on liveness requirements, deterministic behaviour metadata (from Composer), routing dependencies, and user preferences. Signal executes these assignments using dual schedulers: a real-time engine for the live cohort and an anticipative engine running on worker threads. The system dynamically switches nodes between cohorts as the user interacts (e.g., opening plugin UIs, arming tracks), with graph-driven dependency closure ensuring correct propagation of liveness requirements.

For complete details, see [@chorus:/docs/architecture/a06-processing-cohorts-and-anticipative-rendering.md](a06-processing-cohorts-and-anticipative-rendering.md).

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

Composer is accessed over network APIs (e.g. HTTP/HTTPS) and is not part of the core IPC schema defined for Signal/Aura/Pulse.

---

# 7. Real-Time Safety

Any operation that might violate RT constraints MUST NOT occur in the audio callback. This includes:

- memory allocation
- locking
- I/O
- logging
- container resizing

Pulse MUST provide RT-safe immutable graph instructions that Signal can swap atomically.

---

# 8. Evolution Strategy

The architecture is designed for long-term extensibility:

- Plugin sandboxing MAY be added
- Telemetry MAY expand to richer analysis formats
- Remote/UI extensions MAY communicate over network IPC
- Composer MAY gain richer models and vendor integrations

Such changes require new ADRs and spec updates.

---

## 9. Security & Privacy

Loophole's architecture maintains clear trust boundaries:

- **Local IPC**: Signal, Pulse and Aura normally run on the same machine and communicate over local IPC channels. These processes are designed to operate within a trusted local environment.

- **Composer boundary**: Composer is treated as an external intelligence service boundary. No raw audio or user-identifiable data is sent to Composer without explicit user consent. Composer receives only metadata and behavioural signatures, never project content or audio data.

- **Future collaboration**: Collaboration and cloud sync features are planned for future releases and will follow the same privacy-first principles, with explicit user consent and opt-in mechanisms.

- **Data minimisation**: Each component receives only the minimum data necessary for its responsibilities, maintaining clear separation of concerns and minimising attack surface.

---

# 10. Document Conventions

- British English
- Stable heading structure
- All file references use `@repo:/path/` shorthand
- Internal examples use triple-backtick fences
- AI-generated files MUST be wrapped in quadruple backticks

---

# 11. Summary

This document defines the conceptual structure of the Loophole system.

- Chorus defines the architecture.
- Pulse defines the project state.
- Signal defines the audio engine.
- Aura defines the interface.
- Composer defines shared plugin and parameter metadata.

All runtime behaviour MUST conform to the responsibilities defined here.
