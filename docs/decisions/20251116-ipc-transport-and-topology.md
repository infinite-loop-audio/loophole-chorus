# Decision: IPC Transport and Topology

- **ID:** 2025-11-16-ipc-transport-and-topology  
- **Date:** 2025-11-16  
- **Status:** accepted  
- **Owner:** Infinite Loop Audio (Loophole core)  
- **Related docs:**  
  - `docs/specs/ipc/signal/engine.md`  
  - `docs/architecture/01-overview.md`  

---

## 1. Context

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

## 2. Problem Statement

We require a transport that:

1. Works on all target platforms without exotic dependencies.
2. Is straightforward to implement in both C++ and Node.
3. Provides ordered, reliable byte streams.
4. Can support multiple logical channels (commands vs telemetry) if required.
5. Does not introduce heavy protocol complexity (e.g. HTTP/WebSocket framing) where unnecessary.

We also need to decide which process will be the listener and which will
connect, and how this will extend when Pulse becomes a separate service.

---

## 3. Options

### 3.1 Unix Domain Sockets / Named Pipes

**Description**

Use platform-specific local IPC mechanisms: Unix domain sockets on macOS/Linux, named pipes on Windows.

**Pros**

- Slightly lower latency than TCP loopback
- OS-native semantics
- Potentially better performance on some platforms

**Cons**

- Divergent implementations per OS
- Higher complexity early in the project
- Requires platform-specific code paths
- More difficult to test and debug across platforms

**Conclusion**

Rejected. The cross-platform complexity outweighs the performance benefits for initial implementation. A future decision can introduce optional platform-optimised transports if profiling shows it necessary.

---

### 3.2 WebSockets over Loopback

**Description**

Use WebSocket protocol over localhost for IPC communication.

**Pros**

- Easy to integrate with browser environments
- Potential reuse for remote control scenarios
- Well-established protocol

**Cons**

- Extra protocol layers (HTTP upgrade semantics)
- HTTP semantics unnecessary for trusted local IPC
- More overhead and complexity in C++
- Optimised for browser/server scenarios rather than process-local IPC

**Conclusion**

Rejected. The additional protocol complexity is unnecessary for trusted local IPC, and a separate WebSocket-based control plane can be added later if needed.

---

### 3.3 TCP Loopback with Custom Framing

**Description**

Use TCP on the local loopback interface (`127.0.0.1` / `::1`) with a custom length-prefixed framing protocol, carrying structured messages serialised as JSON (initially).

**Pros**

- Portability: works uniformly across macOS, Windows and Linux
- Simplicity: both C++ and Node/Electron have robust, well-tested TCP APIs
- Performance: loopback TCP is sufficiently fast and low latency for DAW-style IPC
- Isolation: binding explicitly to loopback prevents remote access by default
- Future-proofing: can later be extended to remote control or multi-machine setups
- Easy debugging using standard TCP tools
- Natural mapping into JSON Schema and TypeScript interfaces

**Cons**

- TCP loopback is marginally slower than local domain sockets or named pipes
- JSON introduces some encoding/decoding overhead and verbosity
- Port allocation and conflict handling must be managed

**Conclusion**

Accepted. Provides the best balance of simplicity, portability, and performance for initial implementation.

---

## 4. Decision

### 4.1 Transport

Loophole will use:

- **TCP on the local loopback interface** (`127.0.0.1` / `::1`)
- With a **custom length-prefixed framing protocol** on top, carrying
  structured messages serialised as JSON (initially)

Key points:

- No HTTP, WebSockets, or higher-level protocols are used between Aura and Signal.
- Each connection is a reliable, ordered byte stream, framed into discrete messages.
- Message payloads are JSON to begin with, but the framing supports evolution to
  more efficient encodings later (e.g. MessagePack, protobuf, or custom binary).

### 4.2 Topology

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

This will be formalised in a later decision once Pulse is extracted.

### 4.3 Channel Model

Initially:

- A single TCP connection per Loophole session between Aura and Signal.
- Logical separation between:
  - command messages
  - telemetry messages
  - error/diagnostic messages

This separation is implemented at the message level (e.g. `type` fields in the
message envelope), not by multiple sockets.

A future decision may introduce multiple sockets or specialised telemetry channels if
profiling shows it to be beneficial.

---

## 5. Rationale

### 5.1 Why TCP loopback?

- **Portability**: Works uniformly across macOS, Windows and Linux.
- **Simplicity**: Both C++ and Node/Electron have robust, well-tested TCP APIs.
- **Performance**: Loopback TCP is sufficiently fast and low latency for DAW-style IPC.
- **Isolation**: Binding explicitly to loopback prevents remote access by default.
- **Future-proofing**: A TCP-based solution can later be extended to remote control
  or multi-machine setups if desired.

### 5.2 Why not Unix domain sockets or named pipes?

- They provide performance benefits on some platforms, but:
  - Require different implementations per OS.
  - Complicate the initial implementation and testing matrix.
- TCP loopback offers a single cross-platform abstraction.
- If necessary, a future decision can introduce optional platform-optimised transports.

### 5.3 Why not WebSockets?

- WebSockets add HTTP and upgrade semantics that are unnecessary for a trusted,
  local-only engine connection.
- WebSockets are optimised for browser/server scenarios rather than process-local
  IPC.
- A thin TCP protocol is simpler to debug and reason about for engine purposes,
  and a separate WebSocket-based control plane can be added later if needed.

### 5.4 Why JSON payloads initially?

- JSON is easy to inspect and debug.
- It integrates naturally with TypeScript (Aura, Pulse).
- It can be parsed in C++ with mature libraries.
- It allows rapid iteration on message structures in early stages.

Once message schemas stabilise, we can introduce a more efficient encoding
while preserving the same framing and topology.

---

## 6. Consequences

### 6.1 Positive

- Simple, cross-platform IPC implementation.
- Clean separation between engine and UI at the process level.
- Easy debugging using standard TCP tools.
- Natural mapping into JSON Schema and TypeScript interfaces in Chorus.
- Clear path to more efficient encodings without changing transport.

### 6.2 Negative / Trade-offs

- TCP loopback is marginally slower than local domain sockets or named pipes.
- JSON introduces some encoding/decoding overhead and verbosity.
- Port allocation and conflict handling must be managed (e.g. configurable port,
  retry strategy, or ephemeral port discovery).
- Exposing TCP on loopback rather than a more opaque OS-specific mechanism trades
  a slight security advantage for clarity and portability.
- This decision defers remote/network control decisions to a future decision.

---

## 7. Follow-Up Actions

1. Define the exact framing format and message envelope shape in `docs/specs/ipc/`.
2. Specify port configuration and conflict resolution strategy.
3. Create JSON Schema definitions for all IPC message types.
4. Implement TCP server in Signal (C++).
5. Implement TCP client in Aura (TypeScript).
6. Design reconnection and error handling semantics.

---

## 8. Notes

- This decision does not define the message schemas themselves; those will be
  specified in `docs/specs/ipc/`.
- The exact framing format, message envelope shape and port configuration will
  be defined in follow-up specifications.

---
