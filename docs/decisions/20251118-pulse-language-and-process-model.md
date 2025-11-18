# Decision: Pulse Implementation Language & Process Model

- **ID:** 2025-11-18-pulse-language-and-process-model  
- **Date:** 2025-11-18  
- **Status:** accepted  
- **Owner:** Infinite Loop Audio (Loophole core)  
- **Related docs:**  
  - `docs/architecture/03-pulse.md`  
  - `docs/architecture/01-overview.md`  
  - `docs/meta/architecture-backlog.md`  

---

## 1. Context

Loophole's architecture separates the system into four major components:

- **Signal** – C++ audio engine (real-time audio processing, plugins, hardware I/O).  
- **Pulse** – central model/orchestration layer (projects, tracks, nodes, routing, automation, IPC).  
- **Aura** – Electron/TypeScript UI client.  
- **Composer** – external knowledge/metadata service.

Pulse is the **authoritative source of truth** for:

- project state,
- routing & graph configuration,
- interaction with Signal,
- undo/redo,
- persistence and recovery,
- integration with Composer.

Signal and Aura are expected to be **restartable** and **replaceable**:

- If **Signal** crashes: Pulse restarts it and rebuilds the graph.  
- If **Aura** crashes: Pulse continues; Aura reconnects, pulls a snapshot, and resumes.

If **Pulse** crashes, we risk:

- losing project/session state in memory,
- invalidating engine configuration,
- breaking the entire system's invariants.

Therefore, Pulse must be:

- highly robust,
- carefully modelled,
- architected as a separate, supervised process.

We considered multiple implementation strategies and languages for Pulse:

1. **TypeScript/Node** as an in-process library (embedded in Aura).  
2. **TypeScript/Node** as a separate server process.  
3. **Go** as a server process.  
4. **Rust** as a server process.

This decision records the chosen approach.

---

## 2. Problem Statement

We need to choose an implementation language and process model for Pulse that:

1. Supports robust domain modelling for complex project state (tracks, lanes, clips, nodes, routing, automation).
2. Enables strong compile-time invariants to prevent invalid states.
3. Provides predictable performance characteristics without GC pauses that could affect system responsiveness.
4. Aligns with Loophole's long-term positioning as a high-integrity, professional audio tool.
5. Allows Pulse to operate as a separate, supervised process that can restart independently.
6. Facilitates clear IPC contracts with Signal and Aura.

---

## 3. Options

### 3.1 TypeScript/Node, in-process (embedded in Aura)

**Description**

Implement Pulse as a TypeScript library running in the Aura process (Electron renderer), with IPC only between Aura and Signal. Pulse would later be extracted into a separate process if needed.

**Pros**

- Fastest initial implementation.  
- Minimal IPC overhead—direct function calls.  
- Simple dev loop (single process, single codebase).

**Cons**

- Pulse lifetime tied to Aura: any UI crash kills the model.  
- No realistic testing of:
  - process boundaries,
  - reconnection semantics,
  - failure modes.  
- Strong risk of accidental tight coupling between UI and model.  
- Difficult extraction later: function calls must be converted to async IPC.  
- Perception: "core brain is JS embedded in UI".

**Conclusion**

Rejected. This hides important complexity and encourages a monolithic design that conflicts with the long-term architecture.

---

### 3.2 TypeScript/Node, separate Pulse server

**Description**

Implement Pulse as a separate Node process, spawned and supervised by the Electron main process. Aura communicates with Pulse via IPC (e.g. JSON over pipes/WebSocket). Pulse communicates with Signal via the Signal IPC spec.

**Pros**

- Correct **process structure** from day one:
  - Pulse is a separate process,
  - Aura is a client,
  - reconnection and restart paths can be exercised early.
- Implementation speed is relatively high:  
  - TS is familiar,  
  - Node has straightforward IPC,  
  - good integration with Electron.
- Lower barrier to entry for iteration on the model and IPC semantics.

**Cons**

- Runtime is GC-based; pauses are not hard real-time but may cause UX stalls.  
- Type system is weaker than Rust for encoding domain invariants.  
- Long-term perception: "core model is Node.js" — which carries stigma in the pro audio community, even if technically acceptable.  
- Likely requires a rewrite to a systems language later to match Loophole's ambition.

**Conclusion**

Viable technically, but not aligned with long-term positioning and robustness goals.

---

### 3.3 Go server

**Description**

Implement Pulse as a Go server process:

- Goroutine/channel concurrency,
- standard RPC/IPC mechanisms,
- supervised by the Electron main process.

**Pros**

- Very ergonomic concurrency model (goroutines, channels).  
- Well suited for IO-heavy, multi-connection, long-lived servers.  
- Good tooling and deployment model.  
- Perception is stronger than Node for backend work.

**Cons**

- Go's type system is relatively weak for complex domain modelling:
  - no algebraic data types,
  - no pattern matching,
  - more reliance on `interface{}` or `Type` + switch patterns.  
- Complex state invariants must be enforced by convention and tests, not the compiler.  
- Still GC-based (pauses not fatal for Pulse, but not ideal).  
- In the audio/dev tools community, Go is respected but not obviously aligned with high-integrity, DSP-adjacent systems in the same way as Rust/C++.

**Conclusion**

Technically viable and ergonomically pleasant, but mismatched to Pulse's deeply-structured domain model and Loophole's long-term positioning.

---

### 3.4 Rust server

**Description**

Implement Pulse as a Rust server process:

- Async runtime (e.g. `tokio`),
- Strongly typed IPC messages (via `serde`),
- Rich domain modelling (enums, traits, structs),
- Supervised by Electron main process,
- Communicates with Signal and Aura via well-defined IPC.

**Pros**

- Excellent match for Pulse's domain complexity:
  - algebraic data types and pattern matching for tracks/lanes/clips/nodes,
  - compiler-enforced invariants,
  - reduction of invalid states representable in the model.
- No GC; predictable performance characteristics.  
- Strong ecosystem for:
  - async (`tokio`),
  - serialization (`serde`),
  - error handling (`thiserror`, `anyhow`),
  - observability (`tracing`).
- Perception: highly aligned with Loophole's ambition as a serious, long-lived pro tool.
- Clear long-term story:
  - Signal in C++,
  - Pulse in Rust,
  - Aura in TS as a thin client.
- Avoids a future rewrite from Node/Go to Rust.

**Cons**

- Steeper learning curve, especially early in the project:
  - borrow checker,
  - lifetimes,
  - async semantics.  
- Initial development may be slower until patterns are established.  
- Requires tight feedback loop with tooling (Cursor + `cargo`).

**Conclusion**

Best fit for the domain, the long-term vision, and the perception of Loophole as a high-integrity, high-ambition product.

---

## 4. Decision

We will:

1. **Implement Pulse as a separate, supervised server process in Rust**, using an async runtime (`tokio` or equivalent).
2. Define a stable IPC protocol between:
   - **Pulse ⇄ Aura** (for UI commands, snapshots, events),
   - **Pulse ⇄ Signal** (for engine configuration and graph control),
   using the IPC envelope and domain specs defined in Chorus.
3. Implement a **thin TypeScript client** in Aura (`PulseClient`), which:
   - handles connection/reconnection,
   - exposes typed methods corresponding to Pulse IPC commands,
   - treats Pulse as authoritative and stateless from the UI's perspective.
4. Avoid any "temporary" TS/Node implementation of Pulse to prevent architectural drift and future rewrites.

---

## 5. Rationale

This decision was made because:

- **Domain fit**: Rust's type system (algebraic data types, pattern matching, strong ownership) is exceptionally well-suited to Pulse's complex domain model of tracks, lanes, clips, nodes, routing, and automation.
- **Long-term alignment**: Choosing Rust from the start avoids a costly rewrite later and aligns with Loophole's positioning as a high-integrity professional tool.
- **Process architecture**: Implementing Pulse as a separate Rust process from day one ensures we can properly test and validate the process boundaries, reconnection semantics, and failure modes that are critical to the system's robustness.
- **Performance**: No GC means predictable performance characteristics, which is important for a process that orchestrates real-time audio operations.
- **Ecosystem**: Rust's async and IPC ecosystem (`tokio`, `serde`) provides excellent tooling for building a robust server process.

---

## 6. Consequences

### 6.1 Positive

- Pulse can enforce strong invariants in the project model:
  - tracks/lanes/clips/nodes/routing/automation are all robustly typed,
  - invalid combinations can be prevented at compile-time where possible.
- The system will have a correct **process architecture** from day one:
  - Pulse can crash and restart independently,
  - Aura can reconnect and request snapshots,
  - Signal can be restarted and resynchronised by Pulse.
- Loophole's public story is aligned with the architecture:
  - C++ engine,
  - Rust model/orchestrator,
  - TS/Electron UI shell.
- No need for a "big rewrite" from a dynamic language to Rust later.
- Rust ecosystem supports safe concurrency, observability, error handling and IPC.

### 6.2 Negative / Trade-offs

- Initial development pace for Pulse may be slower while:
  - Rust patterns are established,
  - borrow-checker issues are worked through,
  - IPC layers are designed.
- Contributors must be comfortable (or willing to become comfortable) with Rust.  
- Integration cycles must involve tooling (Cursor + `cargo`) to iterate effectively.

### 6.3 Mitigations

- Invest early in a **Pulse Implementation Plan** architecture document:
  - crate layout,
  - IPC transport choices,
  - error handling and logging strategy,
  - testing and snapshot model.
- Leverage:
  - AI-assisted architecture and code generation,
  - Cursor to provide tight compile/run feedback loops.
- Keep Aura's `PulseClient` **very thin** to prevent UI state duplication and lock-in.
- Ensure IPC contracts are defined in Chorus before implementation, so that they remain stable even as Pulse internals evolve.

---

## 7. Follow-Up Actions

1. Create `docs/architecture/XX-pulse-implementation-plan-rust.md`:
   - define crate structure,
   - choose async runtime and IPC stack,
   - outline testing and crash recovery strategies.
2. Ensure all Pulse-related specs in `docs/specs/ipc/pulse/` and Signal IPC docs are compatible with a Rust server implementation.
3. Define the responsibilities and surface of the Aura `PulseClient` TypeScript wrapper.
4. Update any developer onboarding/README docs in Pulse and Aura repositories to reflect this decision.

---

## 8. Notes

- This decision supersedes the earlier assumption in the initial architecture decision that Pulse would be implemented in TypeScript and initially embedded in Aura.
- The Rust implementation will be the first and only implementation of Pulse as a separate process.

---
