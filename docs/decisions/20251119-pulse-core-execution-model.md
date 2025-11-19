# 2025-11-19-pulse-core-execution-model

**ID:** 2025-11-19-pulse-core-execution-model  
**Date:** 2025-11-19  
**Status:** proposed  
**Owner:** Harry  
**Related docs:**
- `docs/architecture/03-pulse.md` (Pulse Architecture)
- `docs/plans/pulse-implementation.md` (Pulse Implementation Plan)
- `docs/decisions/20251118-pulse-language-and-process-model.md` (Pulse Language and Process Model)

---

## Context

Pulse is the central state and orchestration service for Loophole. It owns:

- The authoritative project/model state (`SessionState`)
- All domain logic (project, tracks, clips, routing, etc.)
- IPC handling between Aura (UI) and Signal (engine), now and in future

Pulse will eventually need to support:

- Multiple concurrent inbound IPC messages
- Background tasks (autosave, Composer lookups, hardware polls)
- Potential multi-client scenarios (multiple Aura instances / remote control)
- Tight coordination with Signal’s realtime engine

Rust provides several concurrency primitives (`Arc`, `Mutex`, channels, async runtimes), but the choice of core execution model has a deep impact on:

- How `SessionState` is structured
- How domains are written
- How easy it is to reason about correctness and determinism
- How IPC and background work integrate over time

We need to decide how Pulse’s *core* should own and mutate its state, and where (if anywhere) shared mutable state or locks enter the picture.

---

## Problem Statement

How should Pulse structure its core execution model and state ownership such that:

- `SessionState` is safe, deterministic, and easy to reason about
- Domain logic remains clean and largely oblivious to concurrency concerns
- The architecture can support future concurrency needs (multiple clients, background workers, async IO)
- We avoid premature complexity (e.g. pervasive `Arc<Mutex<...>>`) that makes the code noisier and harder to maintain

In particular:

> Should `SessionState` be wrapped in threadsafe primitives like `Arc<Mutex<SessionState>>` from the outset, or should Pulse core be designed as a single-threaded “actor” that owns state exclusively, with concurrency handled at the boundaries?

---

## Options

### Option A — Actor-style single-threaded core (no `Arc<Mutex<SessionState>>` in core)

Pulse core runs on a single dedicated thread (or logical executor) and owns `SessionState` directly:

```rust
struct PulseCore {
    session: SessionState,
    dispatcher: Dispatcher,
}
```

- All domain handlers receive `&mut SessionState` for each command.
- The core executes a loop:
  - Receive one or more inbound envelopes from a mailbox/queue.
  - Dispatch to domains with `&mut SessionState`.
  - Collect outbound events in an outbox.
  - Release control back to the IO layer.
- Other threads (IPC listener, background workers, async tasks) never touch `SessionState` directly.
- Cross-thread communication is done via channels (command/event queues).

**Pros**

- `SessionState` is simple, owned, and not wrapped in concurrency primitives.
- Domain logic remains clean and deterministic; no locks inside domains.
- Easy to test: Pulse behaves like a pure state machine fed by envelopes.
- Concurrency is localised to the boundaries (IO, background tasks), not baked into the core.
- Future async adoption (Tokio or similar) can be layered around the core without changing domain semantics.
- Lower risk of deadlocks and lock contention.

**Cons**

- Requires designing and maintaining inbound/outbound queues for commands and events.
- If we ever need truly parallel domain execution, it must be orchestrated explicitly (e.g. sharding or parallel command batches), not by default shared-state mutation.
- Slightly more upfront conceptual work for the “core loop + mailbox” pattern.

---

### Option B — Shared-state `Arc<Mutex<SessionState>>` everywhere

`SessionState` is wrapped in `Arc<Mutex<SessionState>>` and shared across threads:

```rust
type SharedSession = Arc<Mutex<SessionState>>;
```

- TCP listener, domain executors, background workers all hold `Arc<Mutex<SessionState>>`.
- Each caller locks the mutex when it needs to mutate state.
- Domain handlers may receive `Arc<Mutex<SessionState>>` or a locked `MutexGuard`.

**Pros**

- Concurrency primitives are explicit and familiar (`Arc<Mutex<...>>`).
- Any thread can mutate `SessionState` “directly” if it has access.
- Easy to spin up worker threads that read/write the same state.

**Cons**

- Domain code becomes littered with locking/unlocking, poisoning concerns, and “how long am I holding this lock?” questions.
- Higher risk of lock contention and deadlocks, especially if IPC handling and domain logic are long-running.
- Harder to reason about determinism and ordering (commands may interleave in complex ways).
- Encourages “reach in and mutate” patterns from multiple places, which works against the clean state-machine model.
- Makes the code noisier and more complex immediately, with little short-term benefit.

---

### Option C — Fully async/await core with shared state

Pulse core is implemented using an async runtime (e.g. Tokio) from the outset. `SessionState` is wrapped in async-aware primitives like `Arc<tokio::sync::Mutex<SessionState>>`, and:

- All domain handlers and IPC functions become `async fn`.
- IO and domain execution are tightly interwoven in a single async task graph.

**Pros**

- Potentially very scalable IO and background work.
- Direct use of async-aware primitives for filesystem/network/composer calls.

**Cons**

- Significantly increases complexity early, before requirements are clear.
- Makes domain and model code care about async for little immediate benefit.
- Async mutexes and structured concurrency add another layer of complexity on top of state management.
- Likely to distract from core model and domain design work that is still in progress.

---

## Decision

We choose **Option A: an actor-style single-threaded Pulse core**, with:

- `SessionState` owned directly by the core (no `Arc<Mutex<SessionState>>` in the inner architecture).
- Domain handlers receiving `&mut SessionState` for each command, as they do today.
- Concurrency and async concerns handled **at the boundaries**:
  - IPC listener(s) and background workers enqueue commands/envelopes into a mailbox/queue.
  - Pulse core processes commands sequentially on a dedicated thread/executor.
  - Outbound events are collected and forwarded back to the IO layer after state mutation.

**Key points:**

- Do **not** wrap `SessionState` in `Arc<Mutex<_>>` within domain or model code.
- Domain/model code must **remain oblivious to concurrency**, only dealing with `&mut SessionState` in a synchronous context.
- Any eventual use of `Arc<Mutex<...>>`, `RwLock`, or async runtimes must be confined to the IPC/server boundary layer, not the core execution model.

Status: **proposed**, to be followed by small documentation updates and implementation alignment.

---

## Rationale

- Pulse is conceptually a **state machine**: given a sequence of commands/envelopes, it should transition from one valid model state to another and emit deterministic events.
- An actor-style core matches this mental model perfectly: a single owner of state, fed by a queue of messages.
- Pushing `Arc<Mutex<SessionState>>` into the core would:
  - pollute domain APIs with locking concerns,
  - encourage uncontrolled shared mutable access,
  - and make testing and reasoning harder, for little benefit at the current stage.
- By keeping `SessionState` as a plain owned struct in the core, we:
  - simplify domain logic,
  - keep lifetime and ownership reasoning straightforward,
  - preserve the option to add concurrency/async at the edges later,
  - and retain deterministic, replayable behaviour (which is valuable for debugging and future testing/replay tools).

This decision aligns with:

- The existing IPC/domain design (domains already take `&mut SessionState` and an outbox).
- The architectural goal of keeping Pulse as a clean, composable service that could be hosted in various runtime environments over time.
- The broader philosophy of Loophole: **strong foundations first**, avoiding premature complexity and “quick-but-messy” patterns that will become liabilities later.

---

## Consequences

**Positive**

- Domain and model code remain simple and synchronous.
- `SessionState` and related model types are easier to reason about and test.
- Concurrency is explicitly represented through message queues and/or outer wrappers, not implicit in the core API.
- We preserve the option to:
  - run the core in its own thread,
  - embed it in different hosting environments,
  - or later introduce async IO without changing domain semantics.
- Locking, if ever introduced, will live in one place (server/IPC boundary), not spread through the codebase.

**Negative / Trade-offs**

- We need to design and implement a small but explicit **core execution loop** and mailbox abstraction.
- If, in future, we decide to parallelise domain workloads (e.g. certain long-running operations or validation steps), this must be explicitly orchestrated rather than “just sharing `SessionState`”.
- Some future contributors might expect `Arc<Mutex<...>>` by default; we must document clearly that this is not the chosen pattern for Pulse core.

---

## Follow-Up Actions

1. **Update Pulse Architecture Doc**
   - Amend `docs/architecture/03-pulse.md` to include:
     - A “Core Execution Model” section describing the actor-style core and single-owner `SessionState`.
     - A note that domain/model code must remain concurrency-agnostic and must not use `Arc<Mutex<_>>` around `SessionState`.

2. **Add a Core Execution Model Architecture Doc**
   - Create a dedicated architecture document (e.g. `docs/architecture/XX-pulse-core-execution-model.md`) that:
     - Describes the core loop (mailbox → dispatch → outbox).
     - Describes how IPC/servers will eventually feed commands into the core.
     - Outlines how background tasks will interact via messages rather than direct state mutation.
     - References this decision document.

3. **Update Pulse Implementation Plan**
   - Extend `docs/plans/pulse-implementation.md` with a section describing:
     - The initial single-threaded core loop.
     - Future steps to introduce a mailbox and possibly a dedicated core thread.
     - Explicitly banning `Arc<Mutex<SessionState>>` in domain/model layers.

4. **Update AGENTS Guidelines**
   - Amend `AGENTS.md` in `loophole-pulse` to:
     - Explicitly state that `SessionState` must not be wrapped in `Arc<Mutex<_>>` inside domains or models.
     - Encourage an actor-style architecture and channel-based concurrency at the edges.

5. **Implementation Alignment**
   - Once the documentation is updated, ensure any new code added to Pulse:
     - Keeps `SessionState` owned directly by the core.
     - Avoids introducing `Arc<Mutex<_>>` in domain/model-level APIs.
     - Structures new features with the expectation of a core message loop.

---

## Notes

- This decision does **not** preclude the use of concurrency, async runtimes, or shared ownership primitives at the Pulse boundary (IPC server, background workers, async filesystem/network calls).
- It simply states that the **core execution model** for Pulse is:
  - single-threaded,
  - actor-style,
  - with a single owner of `SessionState`,
  - and domain logic that remains synchronous and deterministic.
