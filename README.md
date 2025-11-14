<pre>
 ░▒▓██████▓▒░░▒▓█▓▒░░▒▓█▓▒░░▒▓██████▓▒░░▒▓███████▓▒░░▒▓█▓▒░░▒▓█▓▒░░▒▓███████▓▒░
░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░
░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░
░▒▓█▓▒░      ░▒▓████████▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓███████▓▒░░▒▓█▓▒░░▒▓█▓▒░░▒▓██████▓▒░
░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░
░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░
 ░▒▓██████▓▒░░▒▓█▓▒░░▒▓█▓▒░░▒▓██████▓▒░░▒▓█▓▒░░▒▓█▓▒░░▒▓██████▓▒░░▒▓███████▓▒░

  L O O P H O L E - C H O R U S
</pre>

# Overview
Meta-Repository for Architecture, Specifications, ADRs, and Development Protocols

**Chorus** is the meta-repository for the **Loophole** Digital Audio Workstation.

It contains no runtime code.
Instead, it defines the architecture, specifications, protocols, and workflows that bind the Loophole ecosystem into a coherent whole.

If you are working anywhere within the Loophole system, **this is your source of truth**.

---

## Purpose

Chorus provides the authoritative definitions for:

- System architecture
- Data and IPC specifications
- Real-time constraints
- Project-state contracts
- Multi-process boundaries
- Decision records (ADRs)
- AI editing rules and the Meta-Protocol
- Collaboration and task instructions

Every other repository—Signal, Pulse, and Aura—MUST conform to Chorus.

---

## Repository Structure

```
docs/
  architecture/
  specs/
    guidelines/
  decisions/
  meta/
tasks/
.cursor/
```

### Key Directories

- Architecture Overview
  [`@chorus:/docs/architecture/01-overview.md`](docs/architecture/01-overview.md)

- Specifications (IPC, Data Models, Schemas)
  [`@chorus:/docs/specs/`](docs/specs/)

- Decision Records (ADRs)
  [`@chorus:/docs/decisions/`](docs/decisions/)

- Meta-Level Protocols and Editing Rules
  [`@chorus:/docs/meta/`](docs/meta/)

- Task Instructions for Middle Agents
  [`@chorus:/tasks/`](tasks/)

- Cursor Architectural Rules
  [`@chorus:/.cursor/rules/00-architecture.md`](.cursor/rules/00-architecture.md)

---

## The Loophole Ecosystem

Chorus defines the contracts and boundaries between these three runtime repositories:

### Signal — Audio Engine (C++ / JUCE)
Real-time native audio engine.
https://github.com/infinite-loop-audio/loophole-signal

### Pulse — Project and Data Model (TypeScript)
Authoritative project state and modelling engine.
https://github.com/infinite-loop-audio/loophole-pulse

### Aura — User Interface (Electron / TypeScript)
Editing environments, interaction surface, plugin UIs, meters.
https://github.com/infinite-loop-audio/loophole-aura

---

## Getting Started

Recommended reading order:

1. Architecture Overview
   [`@chorus:/docs/architecture/01-overview.md`](docs/architecture/01-overview.md)

2. Specifications (IPC, data structures, schema contracts)
   [`@chorus:/docs/specs/`](docs/specs/)

3. Decision Records
   [`@chorus:/docs/decisions/`](docs/decisions/)

4. Meta-Protocol
   [`@chorus:/docs/meta/meta-protocol.md`](docs/meta/meta-protocol.md)

---

## AI-Assisted Development

Loophole adopts a structured, deterministic approach to AI-assisted editing.

### Core Meta Documents

- Meta-Protocol
  [`@chorus:/docs/meta/meta-protocol.md`](docs/meta/meta-protocol.md)

- Meta-Commands
  [`@chorus:/docs/meta/meta-commands.md`](docs/meta/meta-commands.md)

- Meta Schema
  [`@chorus:/docs/meta/meta-schema.json`](docs/meta/meta-schema.json)

- Cursor Rule Set
  [`@chorus:/.cursor/rules/00-architecture.md`](.cursor/rules/00-architecture.md)

AI tools MUST follow these rules exactly.

---

## Licence

This repository is provided under the **MIT Licence**:

**The Loophole name (including its component names: Signal, Pulse, Aura, Chorus)
may not be used to promote or endorse derived products without prior permission.**

This clause applies to all repositories in the Loophole project.
