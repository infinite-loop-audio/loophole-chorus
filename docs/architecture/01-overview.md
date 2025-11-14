# Loophole Architecture Overview

Loophole is a modular Digital Audio Workstation built as a family of
cooperating processes and repositories:

- **Signal** — low-latency, headless audio engine (C++ / JUCE)
- **Pulse** — project and data model (initially hosted inside Aura)
- **Aura** — rich UI client (Electron / TypeScript)
- **Chorus** — meta-repository for architecture, specs, and cross-repo decisions

The key goals of the architecture are:

- Strong separation of concerns between:
  - Real-time audio processing (Signal)
  - Project/data modelling (Pulse)
  - User experience and interaction (Aura)
- Safe, well-defined IPC between layers
- Plugin isolation and crash resilience
- Freedom to iterate quickly on UI and data model without risking engine stability
- A documentation and tooling strategy that allows AI-assisted development
  (ChatGPT, Cursor, Codex, etc.) to work coherently across repos

## Processes and Responsibilities

### Signal (Audio Engine)

- Owns audio devices and real-time audio callbacks
- Executes an internal audio graph (tracks, buses, plugin chains)
- Hosts third-party plugins (VST3, CLAP, AU)
- Applies sample-accurate parameter and automation changes
- Exposes a minimal IPC surface for:
  - Graph updates
  - Parameter changes
  - Telemetry (meters, transport state, analysis summaries)

Signal deliberately does **not** own high-level project state or plugin taxonomy.

### Pulse (Project / Data Model)

- Owns the authoritative project state:
  - Tracks, clips, routing, buses, sends
  - Plugin instances and their logical identity
  - Automation, markers, tempo map
- Manages undo/redo and state transitions
- Maps user-level concepts (e.g. “Drum Bus Chain”) to concrete engine graph operations
- Initially implemented as a library inside Aura, with a path to becoming an
  independent service process later

Pulse is the source of truth; Signal holds a derived, real-time-optimised view.

### Aura (UI Layer)

- Provides the main user experience:
  - Arrange view
  - Mixer
  - Plugin organisation and discovery
  - Generic plugin editors
  - Visualisation of engine telemetry
- Talks to Pulse to mutate and inspect project state
- Talks to Signal using the IPC schemas defined in Chorus for:
  - Transport control
  - Parameter changes
  - Graph updates
  - Telemetry subscriptions

Aura is where most of the “creative UX” lives, especially around plugin management.

### Chorus (Meta-Repository)

- Defines and maintains:
  - High-level architecture documents (this file and others)
  - IPC and data model specifications
  - Real-time safety guidelines for engine development
  - Cross-repo decisions (ADR records)
  - Task briefs for AI-assisted implementation

Chorus contains **no runtime code**; it is the written contract that the other
repos follow.

## AI-Assisted Development

Loophole is designed to be developed with the help of AI tools:

- ChatGPT (architecture and deep reasoning)
- Cursor (repo-aware implementation and refactoring)
- Codex CLI (precise changes, especially in the C++ engine)

Chorus is the place where architectural intent is written down in a way that
these tools can consume reliably:

- `docs/architecture/` — human-readable design
- `docs/specs/` — machine-friendly schemas and message formats
- `docs/decisions/` — locked-in design choices
- `docs/meta/` — the meta-protocol that describes how specs are updated
- `tasks/` — concrete implementation briefs that tools can act on
