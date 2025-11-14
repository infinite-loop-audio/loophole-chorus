# Glossary

This glossary defines key terms used throughout the Loophole ecosystem.
All documentation, specifications, and ADRs in **Chorus**, as well as related
docs in **Signal**, **Pulse**, and **Aura**, SHOULD use these terms consistently.

This file is normative for terminology.

---

## A

### Architecture
The structural organisation of Loophole into processes, repositories, and
contracts, as defined by the documents in **Chorus**.

### Aura
The **UI layer** of Loophole. An Electron/TypeScript application responsible
for user interaction, visualisation, editing, and high-level control of the
engine and model through IPC.

---

## C

### Chorus
The **meta-repository** for Loophole. Contains architecture documents, IPC
specifications, data model specs, ADRs, the Meta-Protocol, Cursor rules, and
meta-tasks. Chorus contains no runtime code.

---

## D

### DAW
Digital Audio Workstation: an application or system used for recording,
editing, arranging, mixing, and processing audio and MIDI.

---

## I

### IPC (Inter-Process Communication)
The mechanism by which separate processes (e.g. Aura and Signal) exchange
structured messages. In Loophole, IPC is defined by schemas in
`@chorus:/docs/specs/`.

### IPC Contract
A defined, versioned set of messages and semantics that governs communication
between processes. Considered authoritative and binding for implementations.

---

## L

### Loophole
The overall DAW project consisting of:
- Signal (engine)
- Pulse (model)
- Aura (UI)
- Chorus (meta)

---

## M

### Meta-Protocol
The structured mechanism used by AI tools and developers to safely update
documentation and specifications in Chorus using Meta Blocks. Defined in
`@chorus:/docs/meta/`.

### Meta Block
A JSON object, conforming to `meta-schema.json`, that describes a single,
atomic documentation/spec mutation to be applied by a middle agent.

---

## N

### Non-Real-Time (NRT)
Work that does not need to meet real-time deadlines. It can tolerate
scheduling delays, blocking operations, and higher computational complexity.

In Loophole:
- Pulse and Aura execute NRT operations.
- Chorus is entirely NRT.
- NRT work MUST NOT be run on real-time threads in Signal.

---

## P

### Plugin
A third-party or built-in audio or MIDI processing module (e.g. instrument,
effect) hosted inside Signal.

### Pulse
The **project and data model** layer of Loophole. Responsible for project
state, automation, routing, plugin metadata, undo/redo, and deriving engine
graphs for Signal.

### Project State
The complete logical representation of a Loophole project: tracks, routing,
clips, automation, metadata, plugin instances, etc. Pulse is the authoritative
owner of project state.

---

## R

### Real-Time (RT)
Work that must complete within a strict time budget on each audio callback.
RT code must not block, allocate, or perform unpredictable operations.

In Loophole:
- Signal’s audio callback threads are RT.
- Any function reachable from the audio callback is subject to RT constraints.

### Real-Time Safe
A property of code that ensures it conforms to RT constraints (no blocking,
no allocations, no locks, deterministic time complexity).

---

## S

### Signal
The **audio engine** of Loophole. A native C++/JUCE process responsible for
real-time audio processing, plugin hosting, and audio/MIDI I/O.

### Specification (Spec)
A formal, machine-readable or semi-formal definition of data structures,
messages, or behaviours. Specs in Loophole are stored under
`@chorus:/docs/specs/` and are considered binding contracts.

---

## T

### Telemetry
Periodic data emitted from Signal to Aura (and possibly other processes) to
represent meters, analysis results, engine state, and other runtime metrics.

### Task (Chorus Task)
A documented unit of work stored in `@chorus:/tasks/`, usually containing
context, one or more Meta Blocks, and instructions for middle agents.

---

## U

### UI (User Interface)
The graphical and interaction layer presented to the user. In Loophole,
this is provided by Aura.

---

## V

### Versioning (Schema Versioning)
The strategy used to evolve specs, IPC contracts, and data formats while
maintaining compatibility. Governed by documents in
`@chorus:/docs/specs/guidelines/versioning.md`.

---

## W

### Workflow
A repeatable sequence of operations or interactions in the DAW (e.g. plugin
browsing and instantiation, routing setup). Loophole workflows are implemented
primarily in Aura and Pulse.

---

## Notes

This glossary may be extended via ADR-backed changes and the Meta-Protocol.
New terms MUST be added here when they become architecturally relevant.

---
