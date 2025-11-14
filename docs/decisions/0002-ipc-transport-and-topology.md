# ADR 0002 — IPC Transport and Topology

## Status
Accepted

## Date
2025-11-14

---

# 1. Context

Loophole is a multi-process system consisting initially of:

- Aura — UI process (Electron / TypeScript)
- Signal — audio engine process (C++ / JUCE)
- Pulse — model layer (TypeScript, initially embedded in Aura)

These processes must communicate using a well-defined, low-latency, robust IPC
mechanism. This IPC is required for:

- Engine control (transport, graph updates, parameter changes)
- Telemetry (meters, analysis, transport, engine health)
- Project-level commands (eventually via a separate Pulse process)

Constraints:

- Audio engine and UI MUST run in separate processes.
- IPC MUST be reliable and ordered.
- IPC MUST be efficient enough not to compromise real-time performance.
- IPC MUST be cross-platform (macOS, Windows, Linux).
- Implementation MUST be straightforward in both C++ (Signal) and Node/Electron (Aura).
- Future remote control or network features MUST NOT be blocked by this decision.

We need to choose:

- A transport (TCP, Unix domain sockets, named pipes, WebSockets, etc.)
- A basic topology (which side listens, which connects, how many channels)

---

# 2. Problem Statement

We require a transport that:

1. Works on all target platforms without exotic dependencies.
2. Is straightforward to implement in both C++ and Node.
3. Provides ordered, reliable byte streams.
4. Can support multiple logical channels (commands vs telemetry) if required.
5. Does not introduce heavy protocol complexity (e.g. HTTP/WebSocket framing) where unnecessary.

We also need to decide which process will be the listener and which will
connect, and how this will extend when Pulse becomes a separate service.

---

# 3. Decision

## 3.1 Transport

Loophole will use:

- **TCP on the local loopback interface** (`127.0.0.1` / `::1`)
- With a **custom length-prefixed framing protocol** on top, carrying
  structured messages serialised as JSON (initially)

Key points:

- No HTTP, WebSockets, or higher-level protocols are used between Aura and Signal.
- Each connection is a reliable, ordered byte stream, framed into discrete messages.
- Message payloads are JSON to begin with, but the framing supports evolution to
  more efficient encodings later (e.g. MessagePack, protobuf, or custom binary).

## 3.2 Topology

Initial topology:

- **Signal acts as the TCP server** and listens on a loopback address/port.
- **Aura acts as the TCP client** and connects to Signal on startup.
- Aura is responsible for spawning and supervising the Signal process on launch.

This arrangement ensures:

- Signal can be restarted independently if it crashes.
- Aura always knows which engine instance it is connected to.
- The engine is not exposed to the wider network by default.

When Pulse becomes a separate process:

- Pulse will either:
  - connect to Signal via Aura (Aura as broker), or
  - expose its own TCP server on loopback for Aura to connect.

This will be formalised in a later ADR once Pulse is extracted.

## 3.3 Channel Model

Initially:

- A single TCP connection per Loophole session between Aura and Signal.
- Logical separation between:
  - command messages
  - telemetry messages
  - error/diagnostic messages

This separation is implemented at the message level (e.g. `type` fields in the
message envelope), not by multiple sockets.

A future ADR may introduce multiple sockets or specialised telemetry channels if
profiling shows it to be beneficial.

---

# 4. Rationale

### 4.1 Why TCP loopback?

- **Portability**: Works uniformly across macOS, Windows and Linux.
- **Simplicity**: Both C++ and Node/Electron have robust, well-tested TCP APIs.
- **Performance**: Loopback TCP is sufficiently fast and low latency for DAW-style IPC.
- **Isolation**: Binding explicitly to loopback prevents remote access by default.
- **Future-proofing**: A TCP-based solution can later be extended to remote control
  or multi-machine setups if desired.

### 4.2 Why not Unix domain sockets or named pipes?

- They provide performance benefits on some platforms, but:
  - Require different implementations per OS.
  - Complicate the initial implementation and testing matrix.
- TCP loopback offers a single cross-platform abstraction.
- If necessary, a future ADR can introduce optional platform-optimised transports.

### 4.3 Why not WebSockets?

- WebSockets add HTTP and upgrade semantics that are unnecessary for a trusted,
  local-only engine connection.
- WebSockets are optimised for browser/server scenarios rather than process-local
  IPC.
- A thin TCP protocol is simpler to debug and reason about for engine purposes,
  and a separate WebSocket-based control plane can be added later if needed.

### 4.4 Why JSON payloads initially?

- JSON is easy to inspect and debug.
- It integrates naturally with TypeScript (Aura, Pulse).
- It can be parsed in C++ with mature libraries.
- It allows rapid iteration on message structures in early stages.

Once message schemas stabilise, we can introduce a more efficient encoding
while preserving the same framing and topology.

---

# 5. Consequences

## 5.1 Positive

- Simple, cross-platform IPC implementation.
- Clean separation between engine and UI at the process level.
- Easy debugging using standard TCP tools.
- Natural mapping into JSON Schema and TypeScript interfaces in Chorus.
- Clear path to more efficient encodings without changing transport.

## 5.2 Negative

- TCP loopback is marginally slower than local domain sockets or named pipes.
- JSON introduces some encoding/decoding overhead and verbosity.
- Port allocation and conflict handling must be managed (e.g. configurable port,
  retry strategy, or ephemeral port discovery).

## 5.3 Neutral / Trade-offs

- Exposing TCP on loopback rather than a more opaque OS-specific mechanism trades
  a slight security advantage for clarity and portability.
- This ADR defers remote/network control decisions to a future ADR.

---

# 6. Alternatives Considered

### Unix Domain Sockets / Named Pipes

- Pros: Slightly lower latency, OS-native semantics.
- Cons: Divergent implementations per OS; higher complexity early in the project.

### WebSockets over Loopback

- Pros: Easy to integrate with browser environments, potential reuse for remote
  control.
- Cons: Extra protocol layers; HTTP semantics unnecessary for trusted local IPC;
  more overhead and complexity in C++.

### Single-Process Architecture

- Rejected in ADR 0001. Would violate isolation and real-time constraints that
  motivated Loophole’s architecture.

---

# 7. Notes

- This ADR does not define the message schemas themselves; those will be
  specified in `@chorus:/docs/specs/ipc/`.
- The exact framing format, message envelope shape and port configuration will
  be defined in follow-up specifications.

---

# 8. Decision

Loophole will use a **TCP loopback connection** with a simple length-prefixed
message framing protocol and JSON payloads as the default IPC mechanism between
Aura and Signal (and any future local processes), with Signal acting as a local
TCP server and Aura as the client.
