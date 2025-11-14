# ADR 0001 — Initial Multi-Process Architecture

## Status
Accepted

## Context

Loophole aims to provide a DAW with:

- Strong plugin management and organisation
- Rich UI and UX
- Robustness against plugin crashes and instability
- Freedom to iterate rapidly on UI and model layers

A monolithic, single-process DAW architecture (engine + UI + model together)
would make it harder to:

- Isolate plugins and recover from crashes
- Iterate quickly on the UI with web technologies
- Maintain a clear separation between real-time and non-real-time code

## Decision

Loophole is split into three main runtime responsibilities:

1. **Signal** — headless audio engine
2. **Pulse** — project and data model
3. **Aura** — user interface

Additionally, the **Chorus** meta-repository is introduced to hold:

- Cross-repo architecture documentation
- IPC and data model specifications
- Real-time guidelines
- AI-oriented task briefs and meta-protocol

Initially:

- Signal runs as a separate native process.
- Aura hosts Pulse as an internal library.
- Aura communicates directly with Signal over a local IPC channel.

Later:

- Pulse can be migrated into its own service process without breaking the
  contract described in Chorus.

## Consequences

- Signal can remain small and real-time-safe, focused purely on audio.
- UI and project model can evolve quickly with minimal risk to audio stability.
- IPC and data contracts must be carefully designed and maintained.
- Chorus becomes a critical source of truth that must be kept up to date.

This ADR should be updated or superseded if the process model changes in a
substantial way (e.g. introducing additional long-lived services).
