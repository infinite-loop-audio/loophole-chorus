# IPC Semantics Guidelines

This document describes the **conceptual semantics** of Inter-Process
Communication (IPC) in Loophole. It complements the structural definitions in
`@chorus:/docs/specs/ipc/` and is intended to clarify *how* and *when* messages
should be used.

It does not define raw message formats — those live in JSON/TS schemas under
`@chorus:/docs/specs/`.

---

# 1. IPC Participants

Primary IPC participants:

- **Aura → Signal**
  - Control messages (transport, graph updates, parameter changes)
  - Telemetry subscriptions

- **Signal → Aura**
  - Telemetry (meters, analysis, transport state)
  - Error notifications

- **Aura → Pulse** (when Pulse is a separate process)
  - Project state change requests
  - Query requests

- **Pulse → Aura**
  - Project state deltas
  - Validation results

Future participants may include:

- Render/bounce workers
- Plugin sandbox processes
- Remote control surfaces

Each IPC relationship must be documented in `@chorus:/docs/specs/ipc/`.

---

# 2. Message Categories

IPC messages fall into several high-level categories:

## 2.1 Commands

Imperative instructions to perform an action, such as:

- Start/stop transport
- Add/remove/modify graph nodes
- Change parameter values
- Load/unload projects

Commands are typically sent from Aura or Pulse to Signal.

## 2.2 Events

Notifications that something has occurred, such as:

- Engine started/stopped
- Plugin crashed or failed to load
- Transport position changed (if event-driven)

Events are typically emitted by Signal or Pulse.

## 2.3 Queries and Responses

Request/response style messages:

- Requesting project snapshots
- Fetching plugin lists
- Asking for engine capabilities

These are more common between Aura and Pulse than between Aura and Signal.

## 2.4 Telemetry Streams

Regularly emitted messages sent at a fixed or adaptive interval:

- Meter levels
- Spectral data
- Transport status snapshots

Telemetry MUST be structured to avoid overloading IPC channels.

---

# 3. General Semantics

All IPC channels MUST follow these semantic rules:

- Messages MUST be **idempotent** where possible or clearly documented when not.
- Commands targeting specific entities (track, plugin, parameter) MUST include
  stable identifiers (not just indices).
- Messages MUST NOT rely on undocumented ordering guarantees, except where
  a strict ordering is explicitly defined.

All IPC documents MUST describe:

- source process
- destination process
- direction(s)
- required ordering or consistency assumptions

---

# 4. Timing and Ordering

## 4.1 Real-Time vs Non-Real-Time Messaging

- Messages into Signal’s **audio thread** MUST NOT block.
- Signal MUST process commands at deterministic points (e.g. between audio
  blocks or via RT-safe queues).
- Aura and Pulse operate in NRT space and can afford batching/coalescing.

## 4.2 Ordering Guarantees

Unless specified otherwise:

- Messages from Aura to Signal are processed in order of arrival.
- Telemetry messages from Signal to Aura may be:
  - throttled
  - dropped
  - coalesced

Consumers MUST tolerate occasional gaps in telemetry.

## 4.3 Time-Stamped Messages

Certain messages (e.g. automation updates) MAY include explicit timestamps:

- Absolute time (e.g. samples since start)
- Musical time (bars/beats)
- Engine frame index

Schemas for such messages MUST clarify how time is represented.

---

# 5. Reliability and Error Handling

- IPC channels MUST handle unexpected disconnections gracefully.
- On connection loss:
  - Aura SHOULD present a clear error state to the user.
  - Signal SHOULD move to a safe fallback state (e.g. stop audio).
- Error messages MUST:
  - include error codes
  - be machine-readable
  - avoid free-form text-only fields

---

# 6. Backwards Compatibility

IPC semantics MUST be compatible with the versioning strategy in:

```
@chorus:/docs/specs/guidelines/versioning.md
```

Key points:

- New fields SHOULD be optional where possible.
- Deprecation MUST be documented.
- Breaking changes MUST be tied to ADRs.

---

# 7. Observability

IPC should support observability features:

- Ability to log messages for debugging (outside RT threads).
- Ability to inspect current subscriptions and capabilities.
- Optionally, a debug mode for verbose tracing.

This MUST NOT compromise RT safety.

---

# 8. Implementation Notes

- Signal SHOULD use lock-free queues for incoming commands and outgoing
  telemetry relevant to RT operation.
- Aura SHOULD maintain a local representation of UI-facing state, reconciled
  against Pulse and Signal responses.
- Pulse SHOULD treat incoming IPC as a stream of high-level operations, not
  low-level engine mutations.

---

# 9. Future Extensions

Possible future IPC semantics:

- Remote control protocols
- Networked collaboration
- Distributed rendering nodes

Any such extension MUST be introduced via ADR and reflected here.

---

# 10. Summary

This document defines the semantic expectations for IPC in Loophole. Concrete
message structures live in schema files, but the meaning and usage of those
messages must follow the rules and patterns described here.

---
