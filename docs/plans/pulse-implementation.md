# Pulse Implementation Plan

This document describes how to implement **Pulse** as a Rust server process:

- crate and module layout,
- process and concurrency model,
- IPC integration,
- domain structure (project, transport, tracks, nodes, etc.),
- persistence strategy (short-, mid- and long-term),
- testing and validation approach,
- initial milestones.

It is the **“how” companion** to the architecture docs for Pulse and IPC, and
should be kept in sync as we refine implementation details.

---

## 1. Goals & Constraints

Pulse must:

- run as a **standalone Rust server process**, launched by Aura (for now) and
  eventually discoverable / restartable by other tooling.
- own the **authoritative project state**, including:
  - tracks, lanes, clips, nodes, routing,
  - automation and parameters,
  - project metadata, versions and variants,
  - media references (but not media streaming).
- be **single source of truth** for IPC:
  - Aura ↔ Pulse,
  - Pulse ↔ Signal,
  - Pulse ↔ Composer (where applicable).
- be **deterministic** w.r.t. inputs:
  - identical command sequences produce identical state.
- be thoroughly **testable**:
  - model tests,
  - IPC tests,
  - scenario tests.
- be **storage-agnostic** at the core:
  - initial in-memory state,
  - then simple file-based persistence,
  - later, potentially SQLite or similar.

Performance constraints:

- Pulse does *not* run on the audio thread; it has more freedom,
  but still must avoid becoming a bottleneck for UI responsiveness.
- IPC handlers should be efficient, but correctness and clarity are prioritised
  over micro-optimisations initially.

---

## 2. Process & Concurrency Model

### 2.1 Single Authoritative State Thread

Pulse maintains a single **authoritative state** (the “session”) owned by a
“model thread”:

- All mutations of project state happen on this thread.
- Incoming IPC messages are queued and applied sequentially.
- Outgoing events are emitted based on state changes.

This keeps:

- data races out of the core model,
- undo/redo and history management straightforward.

### 2.2 Message Queues

Pulse will have:

- an **incoming command queue**:
  - Aura and Signal push commands (via IPC),
  - the model thread pops and processes them in order.

- an **outgoing event queue**:
  - model thread pushes events when state changes,
  - IPC layer forwards them to subscribers (Aura and Signal).

### 2.3 Background Workers

For heavier tasks, e.g.:

- media analysis coordination,
- building large diffs or snapshots,
- Composer queries,

Pulse may schedule work on background threads or worker pools, but:

- results are always applied back on the model thread,
- the core state remains single-threaded.

---

## 3. Crate & Module Layout

Initial proposal:

- `loophole-pulse/`
  - `Cargo.toml`
  - `src/`
    - `main.rs` — server entrypoint.
    - `lib.rs` — core library exports.
    - `ipc/`
      - `mod.rs`
      - `transport.rs`      — sockets/streams, framing.
      - `envelope.rs`       — matches IPC envelope spec.
      - `dispatcher.rs`     — routes commands to domains.
    - `model/`
      - `mod.rs`
      - `session.rs`        — `SessionState` root.
      - `project/`          — project-level structures.
      - `tracks/`
      - `lanes/`
      - `clips/`
      - `nodes/`
      - `routing/`
      - `automation/`
      - `parameters/`
      - `metadata/`
      - etc.
    - `domains/`
      - `mod.rs`
      - `project_domain.rs`
      - `transport_domain.rs`
      - `tracks_domain.rs`
      - `nodes_domain.rs`
      - `routing_domain.rs`
      - `automation_domain.rs`
      - `parameters_domain.rs`
      - `media_domain.rs`
      - `render_domain.rs`
      - `hardware_domain.rs`
      - `…`
    - `persistence/`
      - `mod.rs`
      - `snapshot.rs`
      - `loader.rs`
      - `saver.rs`
      - `backing_store.rs`  — trait for JSON / SQLite / etc.
    - `tests/` (or `tests/` at repo root)
      - integration tests and scenario tests.

Domains correspond closely to the IPC domain specs and call into `model/*`.

---

## 4. IPC Integration

### 4.1 Transport

Early Pulse builds can use:

- JSON over a simple socket transport:
  - Unix domain sockets on POSIX,
  - named pipes or TCP loopback on Windows.

Key requirements:

- Connection handshake:
  - identify client (Aura, Signal, tool),
  - negotiate protocol version.
- Message framing:
  - length-prefix or delimiter-based framing,
  - robust to partial reads/writes.

The IPC envelope spec from Chorus drives the message structure.

### 4.2 Envelope Mapping

We implement a Rust struct matching the envelope:

```rust
struct Envelope {
    id: String,
    correlation_id: Option<String>,
    source: String,
    target: String,
    domain: String,
    command: String,
    payload: serde_json::Value,
    // plus timestamps, etc., as per spec
}
```

Internally we:

- parse incoming JSON into `Envelope`,
- route based on `domain` + `command`,
- decode `payload` into domain-specific Rust types,
- produce outgoing `Envelope` for events.

The **domain layer** should define strongly-typed payloads; use `serde` to map
between typed structs and `serde_json::Value`.

### 4.3 Domain Dispatcher

A central dispatcher:

```rust
trait DomainHandler {
    fn handle_command(
        &mut self,
        env: &Envelope,
        session: &mut SessionState,
        outbox: &mut Vec<Envelope>,
    );
}
```

We maintain a registry:

```rust
struct Dispatcher {
    handlers: HashMap<String, Box<dyn DomainHandler>>,
}
```

Each domain handler:

- interprets commands for its domain,
- mutates `SessionState`,
- emits zero or more event envelopes.

---

## 5. Core Model: Session & Domains

### 5.1 SessionState

`SessionState` is the root of the in-memory model:

```rust
struct SessionState {
    projects: ProjectRegistry,
    current_project: Option<ProjectId>,
    timebase: TimebaseState,
    transport: TransportState,
    // global maps: plugin registry, devices, etc.
}
```

Projects are loaded into memory one at a time (initially), with room to:

- support multiple open projects later,
- or tabbed/session workflows.

### 5.2 Domain Structure

Each domain:

- has a **pure model** (types and internal APIs),
- an **IPC handler** that translates commands/events to/from that model,
- and may interact with other domains through `SessionState`.

Examples:

- `project_domain` — open/close/save project, saveDraft, snapshots, metadata.
- `transport_domain` — play/stop/seek/loop, tempo follow.
- `tracks_domain` — create/delete tracks, nesting, capabilities.
- `nodes_domain` — NodeGraph editing, channel graphs.
- `routing_domain` — connections between channels, send/return semantics.
- `automation_domain` — curves, lanes, gestures.
- `parameters_domain` — parameter registry and mapping to automation.
- `media_domain` — clip/media references, not actual streaming.
- `render_domain` — offline render requests coordination with Signal.
- `hardware_domain` — devices and mappings (model-side).

Domain logic is pure Rust; no IO inside domain modules.

---

## 6. Persistence Strategy (Implementation View)

The architecture already defines a **flexible persistence strategy**. The
implementation plan is:

### 6.1 Phase 1 — In-Memory Only

- Initial Pulse builds:
  - no actual on-disk persistence,
  - project is ephemeral across runs.
- Useful for:
  - validating IPC contracts,
  - testing basic editing,
  - integrating Aura and Signal flows.

### 6.2 Phase 2 — Simple File-Based Save/Load

- Implement JSON (or similar) serialisation of:
  - `SessionState` → `ProjectSnapshot`,
  - `ProjectSnapshot` ↔ on-disk representation.
- Save/load flows for:
  - `project.save`
  - `project.open`
  - `project.saveDraft`
- Atomic writes:
  - write to temp,
  - fsync/rename to final name.

### 6.3 Phase 3 — Embedded Database (e.g. SQLite)

- Introduce a `BackingStore` trait, with implementations for:
  - file-based JSON,
  - SQLite-based store.
- Transition to SQLite (if chosen) for:
  - project metadata,
  - version history,
  - potentially fine-grained edit logs.

This allows Pulse to adopt SQLite without changing the core model:

```rust
trait BackingStore {
    fn load_project(&self, path: &ProjectPath) -> Result<ProjectSnapshot>;
    fn save_project(&self, path: &ProjectPath, snapshot: &ProjectSnapshot) -> Result<()>;
}
```

A separate decision doc will formalise the concrete persistence strategy.

---

## 7. Testing & Validation

Implementation should start with testing baked in.

### 7.1 Unit Tests

- Model types:
  - tracks, clips, nodes, lanes,
  - routing graphs,
  - automation curves,
  - parameters.

- Domain logic:
  - each domain module has tests for:
    - command handling,
    - state transitions,
    - validation rules.

### 7.2 IPC & Integration Tests

- Round-trip tests:
  - encode/decode envelopes,
  - domain payload validation.

- Scenario tests:
  - create project → add tracks → add clips → route nodes → save,
  - reload project and compare model equality.

- “Fake client” tests:
  - small harness that simulates Aura and Signal clients
    sending IPC commands over a loopback connection.

### 7.3 Non-Functional Checks

- Performance baselines:
  - initial perf tests for large projects (many tracks/clips/nodes).
- Consistency checks:
  - determinism of command sequences,
  - invariants (e.g. no orphaned nodes, valid routing graphs).

---

## 8. Initial Milestones

### 8.1 Milestone P0 — Pulse Skeleton Server

- Rust crate layout created (`loophole-pulse`).
- `main.rs` starts a server:
  - binds to local socket/pipe,
  - accepts connections from one client (Aura prototype).
- Envelope parsing and validation:
  - basic ping/pong or `project.hello` command.
- In-memory `SessionState` with:
  - empty project registry,
  - default timebase and transport state.

Success criteria:

- CLI can start Pulse,
- Aura (or a simple client) can:
  - connect,
  - send a test command,
  - receive a structured response.

### 8.2 Milestone P1 — Project & Transport Domains

- Implement:
  - `project.open`, `project.saveDraft`, `project.save`,
  - `transport.play`, `transport.stop`, `transport.seek`.
- Simple, in-memory project representation.
- Minimal unit tests for project + transport state transitions.
- IPC integration tests for the above commands.

### 8.3 Milestone P2 — Tracks, Lanes & Channels (Skeleton)

- Add model types for:
  - tracks,
  - lanes,
  - channels (graph-level ID only at this stage).
- Implement basic commands:
  - `tracks.addTrack`, `tracks.removeTrack`,
  - simple nesting/folder semantics.

No Signal integration yet beyond stubs; focus on Pulse model.

### 8.4 Milestone P3 — NodeGraph & Routing Skeleton

- Introduce NodeGraph model matching the Node architecture.
- Basic operations:
  - add/remove nodes,
  - connect/disconnect routing.
- IPC commands to exercise these operations.

Further milestones tie into Signal and Composer once the Pulse core is stable.

---

## 9. Tooling & Developer Experience

For Pulse development:

- Use `cargo test` as the primary test runner.
- Use `cargo fmt` and `cargo clippy` to enforce style and linting.
- Provide a small “dev harness” binary:
  - e.g. `pulse-cli` that can:
    - start the server,
    - send test commands,
    - pretty-print responses.

Cursor can be used to:

- scaffold modules from this plan,
- generate initial Rust types from IPC specs,
- write tests from the described scenarios.

This document should be revisited after Milestone P0 and P1 are complete to
reflect real-world implementation experience and refine the plan.
