# Real-Time Safety Guidelines

This document defines required constraints and patterns for **real-time (RT)**
code in Loophole, primarily within the **Signal** audio engine.

It is normative:
All real-time code in Signal MUST follow these rules.

---

# 1. Real-Time Constraints

In RT contexts (e.g. audio callbacks), code MUST:

- Complete within a strict time budget per audio block
- Avoid all forms of blocking, waiting, or unbounded work
- Execute deterministically (time complexity O(n) where n = buffer size)

---

# 2. Prohibited Operations in RT Code

The following operations MUST NOT occur in RT code:

- Memory allocation and deallocation:
  - `new`, `delete`, `malloc`, `free`
  - STL containers that may trigger allocation (`std::vector::push_back`, etc.)
- Locks and synchronisation primitives:
  - `std::mutex`, `std::lock_guard`, `std::unique_lock`
  - Condition variables, semaphores
- I/O and OS calls:
  - Disk access
  - Network access
  - Console logging, file logging
  - Plugin scanning or dynamic loading
- Non-constant-time algorithms:
  - Hash map insertion/rehashing
  - Tree rebalancing
  - Sorting on unbounded collections

---

# 3. Allowed Patterns

RT-safe code MAY:

- Read from and write to pre-allocated buffers
- Use lock-free structures designed for RT usage
- Use atomic operations with care (e.g. `std::atomic<T>`)
- Read configuration values set from non-RT threads
- Process audio in fixed, precomputed graphs

---

# 4. Communication Between RT and Non-RT

Signal SHOULD:

- Use **single-producer, single-consumer lock-free queues** for:
  - Incoming commands from Aura/Pulse
  - Outgoing telemetry from RT thread to non-RT
- Apply commands at RT-safe points (e.g. between processing blocks)

Non-RT threads MAY:

- Construct new graphs
- Prepare plugin instances
- Compute analysis results on copies of audio data

---

# 5. Plugin Considerations

Plugins hosted by Signal:

- MUST be treated as untrusted code
- MAY allocate internally (outside Loophole’s control)
- SHOULD be isolated via:
  - Defensive coding around edges
  - Optional sandboxing processes in future ADRs

Signal code surrounding plugins MUST remain RT-safe even if plugins misbehave.

---

# 6. Testing and Validation

Signal SHOULD include tests and diagnostics to:

- Detect violation of RT invariants (e.g. allocation counters in debug builds)
- Measure worst-case processing times for typical workloads
- Verify that graph edits occur outside RT callbacks

Documentation in Signal SHOULD link back to this file.

---

# 7. Design Guidance for Signal

When designing new features:

- Prefer **precomputed, immutable structures** used by the RT thread
- Treat the RT callback as a pure function:
  - Inputs: audio, MIDI, precomputed graph, parameter streams
  - Output: processed audio, telemetry metrics
- Move complex logic into:
  - preparation stages
  - model layer (Pulse)
  - analysis workers

---

# 8. Coordination With Pulse and Aura

- Pulse MUST provide graph updates in a form that can be applied cheaply in RT.
- Aura MUST NOT attempt to drive Signal directly with complex, high-frequency
  changes without going through proper RT-safe pipes.

---

# 9. Summary

These guidelines ensure Signal remains robust, stable, and glitch-free, even
under heavy load and complex projects. RT safety is non-negotiable and MUST
be preserved in all engine changes.

---
