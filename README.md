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

# Meta-Repository for Loophole Architecture and Specifications

Chorus is the meta-repository that defines the architecture, specifications,
protocols, and decision records for the Loophole Digital Audio Workstation.

It contains no runtime code. All other repositories in the Loophole ecosystem
(Signal, Pulse and Aura) must conform to the documents stored here.

---

## Contents

- [Purpose](#purpose)
- [Repository Structure](#repository-structure)
- [The Loophole Ecosystem](#the-loophole-ecosystem)
- [Getting Started](#getting-started)
- [AI-Assisted Development](#ai-assisted-development)
- [Licence](#licence)

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

Every other repository—Signal, Pulse, Aura—must conform to Chorus.

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
  [`@chorus:/docs/architecture/a01-overview.md`](docs/architecture/a01-overview.md)

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

Chorus defines the architectural and contractual relationships between the three
runtime repositories:

### Signal — Audio Engine (C++ / JUCE)
https://github.com/infinite-loop-audio/loophole-signal

### Pulse — Project and Data Model (TypeScript)
https://github.com/infinite-loop-audio/loophole-pulse

### Aura — User Interface (Electron / TypeScript)
https://github.com/infinite-loop-audio/loophole-aura

---

## Getting Started

Recommended reading order:

1. Architecture Overview
   [`@chorus:/docs/architecture/a01-overview.md`](docs/architecture/a01-overview.md)

2. Specifications (IPC, structures, schema contracts)
   [`@chorus:/docs/specs/`](docs/specs/)

3. Decision Records
   [`@chorus:/docs/decisions/`](docs/decisions/)

4. Meta-Protocol
   [`@chorus:/docs/meta/meta-protocol.md`](docs/meta/meta-protocol.md)

---

## AI-Assisted Development

Loophole uses a structured and deterministic approach to AI-assisted editing.

### Core Meta Documents

- Meta-Protocol
  [`@chorus:/docs/meta/meta-protocol.md`](docs/meta/meta-protocol.md)

- Meta-Commands
  [`@chorus:/docs/meta/meta-commands.md`](docs/meta/meta-commands.md)

- Meta Schema
  [`@chorus:/docs/meta/meta-schema.json`](docs/meta/meta-schema.json)

- Cursor Rule Set
  [`@chorus:/.cursor/rules/00-architecture.md`](.cursor/rules/00-architecture.md)

AI tools must follow these documents precisely.

---

## Licence

This repository is provided under the MIT Licence with the following additional clause:

**The Loophole name (including its components: Signal, Pulse, Aura and Chorus)
may not be used to promote or endorse any derived product without prior written
permission from the copyright holder.**

This clause applies to all repositories within the Loophole ecosystem.
