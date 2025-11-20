# Client Lifecycle, Identity, and Heartbeat Protocol

**ID:** 2025-11-20-client-lifecycle-and-heartbeat  
**Date:** 2025-11-20  
**Status:** accepted  
**Owner:** Loophole Architecture Group  
**Related docs:**
- @chorus:/docs/specs/ipc/envelope.md
- @chorus:/docs/specs/ipc/pulse/client.md
- @pulse:/docs/architecture/b02-ipc-and-session-core.md
- @aura:/docs/architecture/a01-overview.md

---

## Context

Loophole’s runtime architecture is split into multiple long-running processes:

- **Pulse** — the central session and project server.
- **Aura** — the primary UI client (Electron + Svelte).
- **Future UI clients** — satellite or alternative UIs that speak the same IPC protocol.

The **client domain** in the IPC specification is responsible for:

- identifying UI clients to Pulse,
- tracking which client (if any) “manages” the Pulse process,
- maintaining liveness (heartbeats),
- supporting clean shutdown and reconnection.

We recently introduced:

- `clientId` and `instanceId` in the client handshake,
- manager semantics (which client is allowed to *control* Pulse’s lifecycle),
- a heartbeat mechanism for liveness,
- the `kind` + `name` envelope conventions.

These behaviours were partially implemented but not yet captured as a formal decision.

This ADR records the canonical model for:

- client identity,
- manager reclamation,
- heartbeat direction,
- and request/response/event naming.

---

## Problem Statement

We need a robust and extensible way for Pulse to:

1. **Identify UI clients**, even across crashes and reconnects.
2. **Elect and track a single “manager” client**, which may be responsible for starting and stopping Pulse (e.g. Aura in desktop mode).
3. **Detect dead / disconnected clients**, without relying solely on TCP socket state.
4. **Support future multi-client UI scenarios** (secondary control surfaces, remote UIs) without rewriting the client domain.
5. **Keep the IPC envelope model coherent**, particularly:
   - how `kind` and `name` interact,
   - how commands, responses and events are named.

The initial implementations had a number of issues:

- Ambiguity between `clientId` and `instanceId`.
- Confusing “managed / manager” semantics.
- Heartbeat direction unclear (who pings whom?).
- Inconsistent naming between:
  - commands,
  - responses,
  - and events using similar names.

We need a single, clear model that:

- matches the emerging implementation in Pulse and Aura,
- is easy to reason about,
- and scales to future clients.

---

## Options

### Option A — Manager tied to clientId only, heartbeat from Pulse, strict naming rules

**Identity and manager semantics**

- Each UI client sends:
  - `clientId`: stable identifier of the client *kind* (e.g. `"aura"`).
  - `instanceId`: unique identifier of this running instance.
- Pulse stores:
  - `managerClientId: string | null`.
  - `managerInstanceId: string | null` (informational only).
- Manager election:
  - On startup, Pulse may be passed `--client-id` and `--client-instance-id`.
  - These become the initial `managerClientId` / `managerInstanceId`.
- On each `client.hello`:
  - If `incoming.clientId == managerClientId`, Pulse sets `isManager = true` for that client and updates `managerInstanceId`.
  - Otherwise `isManager = false`.
  - Only one client is treated as manager at any given time.

**Heartbeat**

- Pulse periodically sends a `client.heartbeat` **command** to each connected client.
- Clients respond with a `client.heartbeat` **response**.
- Missing responses/timeout → Pulse marks the client as dead and may emit events.

**Naming**

- **Commands and responses share the same `name`.**
  - e.g. `client.hello` command, `client.hello` response.
- **Events use different names**:
  - e.g. `client.connected`, `client.disconnected`, `client.timedOut`.

**Pros**

- Simple, stable identity model.
- Manager reclamation “just works” after an Aura crash:
  - new Aura instance with same `clientId` automatically becomes manager.
- Clear, symmetric request/response semantics (`kind` differentiates direction).
- Events are semantically distinct; no ambiguity with request/response.
- Scales naturally to multiple UI clients; only one manager at a time.

**Cons**

- Requires careful implementation of timeouts and heartbeat scheduling in Pulse.
- Slightly more boilerplate in the client domain (command + response handlers).

---

### Option B — Manager tied to (clientId, instanceId) pair

**Identity and manager semantics**

- The manager is strictly the **same instance** that started Pulse.
- Pulse compares both `clientId` *and* `instanceId` to decide `isManager`.

**Pros**

- Strong guarantee that only the original instance can be manager.

**Cons**

- If Aura crashes, a new instance **can never** reclaim management of Pulse.
- Leads to orphaned Pulse processes and confusing UX.
- Requires external tooling or manual user intervention to repair.

---

### Option C — No explicit manager semantics, “whoever connects first wins”

**Identity and manager semantics**

- Pulse does not track a formal “manager”.
- The first UI client to connect implicitly “owns” the process.

**Pros**

- Very simple to implement initially.

**Cons**

- Not explicit, not robust, and does not support:
  - headless Pulse,
  - remote controllers,
  - or future compositions with other UIs.
- Hard to document and reason about in multi-client scenarios.

---

## Decision

We choose **Option A**.

Concretely:

1. **Client identity**
   - Each UI client sends in `client.hello`:
     - `clientId: string` — stable identifier of the client kind (e.g. `"aura"`).
     - `instanceId: string` — unique identifier of this running instance.
   - Pulse tracks:
     - `managerClientId: string | null`.
     - `managerInstanceId: string | null`.

2. **Manager reclamation**
   - Pulse may be started with:
     - `--client-id <clientId>`
     - `--client-instance-id <instanceId>`
   - These seed `managerClientId` / `managerInstanceId`.
   - On every `client.hello`:
     - If `incoming.clientId == managerClientId`, that client is flagged `isManager = true` and `managerInstanceId` is updated.
     - Otherwise `isManager = false`.
   - This allows a new instance of Aura (or any client with the same `clientId`) to reclaim management of Pulse after a crash.

3. **Heartbeat protocol**
   - Pulse periodically sends:
     - `domain: "client"`
     - `kind: "command"`
     - `name: "heartbeat"`
   - Each UI client **must** respond with:
     - `domain: "client"`
     - `kind: "response"`
     - `name: "heartbeat"`
   - Clients that fail to respond within a configured timeout are marked as unhealthy or disconnected; Pulse may emit events such as `client.timedOut` or `client.disconnected`.

4. **Naming conventions**
   - **Commands and responses share the same `name`**:
     - `client.hello` (command) / `client.hello` (response).
     - `client.heartbeat` (command) / `client.heartbeat` (response).
   - **Events always use distinct names**, never reusing a command name:
     - e.g. `client.connected`, `client.disconnected`, `client.timedOut`.
   - `kind` is the primary discriminator for direction and semantics:
     - `command` — request from client to Pulse, or from Pulse to client.
     - `response` — direct reply to a previous command (matched via `cid`).
     - `event` — unsolicited notification.

---

## Rationale

- **Robust recovery after crashes.**  
  By tying manager status to `clientId` only, a new instance of a given client can reclaim control of Pulse without manual intervention. This is essential for Aura, which is expected to be restarted frequently during development and possibly in production.

- **Clear separation of concerns.**  
  Pulse decides which client is manager. Aura simply provides identity and behaves according to `isManager` in the handshake response.

- **Symmetric and predictable protocol.**  
  Using the same `name` for command/response pairs and reserving distinct names for events makes the protocol easy to understand and introspect. `kind` becomes the reliable indicator of direction.

- **Future multi-client support.**  
  The model is compatible with:
  - multiple UI clients connected simultaneously,
  - future secondary UIs (e.g. remote transport, control surfaces),
  - potential headless or service-style clients.

- **Extensibility.**  
  Heartbeats and manager status can later feed:
  - diagnostics UIs,
  - connection graphs,
  - health dashboards,
  - and more advanced scheduling (e.g. preferring a particular client to remain manager).

---

## Consequences

### Positive

- Aura can safely crash and restart without leaving Pulse orphaned.
- The architecture explicitly supports “one manager, many clients”.
- Observability and debugging become easier:
  - `clientId`, `instanceId`, and `isManager` can be surfaced via diagnostics.
- The naming rules for commands/responses/events generalise across all IPC domains.

### Negative / Costs

- Requires implementation work in:
  - Pulse’s client domain:
    - manager tracking,
    - heartbeat scheduling,
    - timeout handling,
    - event emission.
  - Aura’s session/pulse domain:
    - hello/heartbeat responses,
    - surfacing `isManager`.
  - Chorus IPC docs:
    - to align the client spec with the chosen semantics.
- Slightly more complexity in client startup:
  - Clients must generate and persist `instanceId` appropriately.
  - Aura must decide when to reuse vs regenerate `instanceId` (e.g. per run vs per install).

---

## Follow-Up Actions

1. **Chorus IPC spec**
   - Update `@chorus:/docs/specs/ipc/pulse/client.md` to:
     - use `clientId` + `instanceId` terminology,
     - describe the manager model based on `clientId`,
     - define `client.hello` and `client.heartbeat` as command/response pairs with shared `name`,
     - formalise event naming: `client.connected`, `client.disconnected`, `client.timedOut`, etc.

2. **Pulse**
   - Update the client domain implementation to:
     - honour the new handshake structure,
     - track `managerClientId` / `managerInstanceId`,
     - set `isManager` based on `clientId` equality,
     - implement heartbeat command broadcast and response tracking,
     - emit appropriate client lifecycle events.

3. **Aura**
   - Ensure Aura:
     - sends `client.hello` with `clientId="aura"` and a generated `instanceId`,
     - stores and surfaces `isManager` from the handshake response,
     - responds to `client.heartbeat` commands with `client.heartbeat` responses,
     - exposes minimal client/manager/heartbeat state in the session domain for debug UIs.

4. **Diagnostics**
   - Extend diagnostics UIs (in Aura) to show:
     - connected clients,
     - `clientId`, `instanceId`,
     - `isManager`,
     - last heartbeat timestamps.

---

## Notes

- This ADR applies specifically to **UI clients** (Aura and future equivalents).  
  Signal and Composer will communicate with Pulse through other, more specialised protocols and are **not** considered UI clients in this sense.
- If, in future, we need to allow multiple managers (e.g. cluster or failover modes), we will revisit this ADR or supersede it with a more advanced model.
