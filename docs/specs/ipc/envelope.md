# IPC Envelope Specification

This document defines the common envelope format used by all Loophole IPC
messages between Aura, Pulse, Signal and Composer.

The envelope provides:

- a consistent structure for all JSON-based IPC messages,
- a clear routing model centred on Pulse,
- support for domain-based message classification,
- correlation for request/response pairs,
- priority hints for scheduling,
- a foundation for binary fast-path streams (gestures, metering, etc.),
- a forwards-compatible versioning scheme.

Domain-specific message payloads (Project, Transport, Track, Clip, etc.) are
documented separately. This document only describes the shared envelope.

---

## Contents

- [1. Goals](#1-goals)  
- [2. Envelope Types](#2-envelope-types)  
- [3. JSON Envelope Structure](#3-json-envelope-structure)  
  - [3.1 Fields](#31-fields)  
- [4. Message Semantics and Routing](#4-message-semantics-and-routing)  
  - [4.1 Central Routing Model](#41-central-routing-model)  
  - [4.2 Commands, Events, Snapshots, Responses and Errors](#42-commands-events-snapshots-responses-and-errors)  
- [5. Ordering and Reliability](#5-ordering-and-reliability)  
  - [5.1 Per-Connection Ordering](#51-per-connection-ordering)  
  - [5.2 Delivery Semantics](#52-delivery-semantics)  
- [6. Binary Fast-Path Frames](#6-binary-fast-path-frames)  
  - [6.1 Binary Frame Header (Conceptual)](#61-binary-frame-header-conceptual)  
  - [6.2 Stream Setup via JSON](#62-stream-setup-via-json)  
- [7. Error Model](#7-error-model)  
- [8. Versioning Strategy](#8-versioning-strategy)  
- [9. Security and Safety Notes](#9-security-and-safety-notes)  
- [10. Examples](#10-examples)  
  - [10.1 Aura → Pulse: User Pressed Play](#101-aura--pulse-user-pressed-play)  
  - [10.2 Pulse → Signal: Cohort Graph Update](#102-pulse--signal-cohort-graph-update)  
  - [10.3 Pulse → Aura: Project Snapshot](#103-pulse--aura-project-snapshot)

---

## 1. Goals

The IPC envelope format must:

- work across all logical links:

  - Aura ⇄ Pulse  
  - Pulse ⇄ Signal  
  - Pulse ⇄ Composer  

- support domain-based routing (project, transport, track, clip, etc.),
- make Pulse the central authority and router,
- clearly separate:

  - message metadata (routing, IDs, timing, priority),  
  - message payload (domain-specific content),  
  - high-rate binary streams (gestures, metering),

- be versioned and forwards-compatible,
- support:

  - request/response pairs (via correlation IDs),  
  - fire-and-forget events,  
  - snapshots vs incremental updates,  
  - high-rate streams via a dedicated binary path.

---

## 2. Envelope Types

There are two high-level envelope families:

1. **JSON envelopes**  
   Used for all normal IPC:

   - commands,
   - events,
   - snapshots,
   - responses,
   - errors.

2. **Binary frames**  
   Used for high-rate streams:

   - gesture/control streams,
   - metering,
   - (future) waveform or spectral chunks.

JSON envelopes carry structured domain messages. Binary frames carry
time-sensitive data with minimal overhead.

---

## 3. JSON Envelope Structure

Every normal IPC message is a single JSON object with this shape:

```jsonc
{
  "v": 1,
  "id": "msg-uuid-or-monotonic-id",
  "cid": "correlation-id-or-null",
  "ts": "2025-11-17T12:34:56.789Z",

  "origin": "aura|pulse|signal|composer",
  "target": "aura|pulse|signal|composer",

  "domain": "project|transport|track|clip|lane|channel|node|parameter|automation|routing|mixer|ui|engine|cohort|...",

  "kind": "command|event|snapshot|response|error",

  "name": "open|snapshot|play|create|add|...",

  "priority": "realtime|high|normal|low",

  "payload": { },

  "error": {
    "code": "string-code",
    "message": "human-readable",
    "details": { }
  }
}
```

### 3.1 Fields

- `v` (number)  
  Envelope schema version. Starts at `1`.  
  Governs the top-level fields of the envelope itself (not domain payloads).

- `id` (string)  
  Unique message ID.  
  - Must be unique per connection (UUID or connection-scoped monotonic ID).
  - Used for logging, tracing and debugging.

- `cid` (string or null)  
  Correlation ID.  
  - For request/response pairs:

    - request: `cid = null`,  
    - response: `cid = id` of the original request.

  - Fire-and-forget messages (most events) use `null`.

- `ts` (string, ISO 8601, required)  
  Timestamp when the message was created by the origin.  
  - **Required**: The sender must include a timestamp. This field must never be null or omitted.
  - Must be a valid ISO 8601 timestamp string (e.g. `"2025-11-17T12:34:56.789Z"`).
  - If an implementation auto-fills this field, it must still be an ISO 8601 string, never null.
  - Used for ordering diagnostics and drift detection.
  - Not intended for sample-accurate audio timing (that is a domain concern).

- `origin` (string)  
  Origin of the message. One of:

  - `"aura"`, `"pulse"`, `"signal"`, `"composer"`.

- `target` (string)  
  Intended recipient of the message. Same set as `origin`.  
  In practice, Pulse is the central hub; direct Aura ⇄ Signal or Signal ⇄ Composer
  messaging is not permitted.

- `domain` (string)  
  Domain namespace for the message. Initial domains include:

  - `"project"`, `"transport"`, `"timebase"`,  
  - `"track"`, `"clip"`, `"lane"`, `"channel"`,  
  - `"node"`, `"parameter"`, `"automation"`,  
  - `"routing"`, `"mixer"`,  
  - `"media"`, `"recording"`, `"rendering"`,  
  - `"gesture"`, `"metering"`,  
  - `"ui"`, `"pluginUi"`, `"pluginLibrary"`,  
  - `"engine"`, `"cohort"`, `"session"`,  
  - `"history"`, `"hardwareIo"`, `"engineDiagnostics"`, `"projectMetadata"`.

  Domain-specific documents define the available names and payloads within each
  domain.

- `kind` (string)  
  High-level category of message:

  - `"command"` – request to perform an action.
  - `"event"` – notification that something happened (fire-and-forget).
  - `"snapshot"` – authoritative state projection from Pulse to Aura.
  - `"response"` – response to a command.
  - `"error"` – error response or error event.

- `name` (string)  
  Unqualified message name within the domain. Examples:

  - `"open"`, `"snapshot"`,  
  - `"play"`, `"stateChanged"`,  
  - `"create"`, `"split"`,  
  - `"add"`, `"setValue"`,  
  - `"cohortAssigned"`.

  The fully-qualified label is derived as `domain + "." + name` (e.g. `"project.open"`). This fully-qualified form is available to runtimes that want it for logging or debugging, but is not part of the envelope structure itself.

  **Naming Conventions:**
  - **Commands and responses share the exact same `name`**, differentiated only by `kind`. For example, a `client.hello` command (kind="command") and its response (kind="response") both use `name: "hello"`.
  - **Events must never reuse a command name**. Events use distinct names such as `"connected"`, `"disconnected"`, `"stateChanged"` that do not conflict with command names.
  - The `name` field must not contain domain separators (e.g. `"gesture.begin"` is invalid; use `domain: "gesture"`, `name: "begin"` instead).

  Each domain spec enumerates the valid names and their payloads.

- `priority` (string)  
  Hint for scheduling and queueing:

  - `"realtime"` – as time-critical as possible (lightweight engine status, certain control signals).
  - `"high"` – user input and critical edits.
  - `"normal"` – general traffic.
  - `"low"` – background updates, non-urgent tasks.

  This field is advisory; implementations may use it to create separate queues.

- `payload` (object)  
  Domain-specific content. Each `name` (within its `domain`) has its own schema defined in the
  relevant domain document.

- `error` (object or null)  
  Present only when `kind = "error"` (or for specific error-type messages).
  Fields:

  - `code` – machine-friendly error code (e.g. `"project.notFound"`).
  - `message` – short human-readable description.
  - `details` – optional, domain-specific data (validation errors, paths, etc.).

---

## 4. Message Semantics and Routing

### 4.1 Central Routing Model

Pulse is the central authority and router.

Logical flows:

- Aura → Pulse  
  - `origin = "aura"`, `target = "pulse"`  
  - Editing intents, transport commands, UI-driven actions.

- Pulse → Aura  
  - `origin = "pulse"`, `target = "aura"`  
  - Snapshots, validation errors, notifications.

- Pulse → Signal  
  - `origin = "pulse"`, `target = "signal"`  
  - Graph updates, cohort assignments, parameter/automation streams.

- Signal → Pulse  
  - `origin = "signal"`, `target = "pulse"`  
  - Engine status, plugin failures, DSP-side notifications.

- Pulse → Composer / Composer → Pulse  
  - Metadata queries and telemetry, never project-critical state.

Direct messaging paths such as Aura → Signal or Signal → Composer are not
permitted. All cross-layer communication flows through Pulse.

### 4.2 Commands, Events, Snapshots, Responses and Errors

- `kind = "command"`  
  - A request to perform an action.
  - Typically sent from Aura → Pulse or Pulse → Signal.
  - May expect a `response`. Domain specs define which commands expect responses
    and what form they take.

- `kind = "response"`  
  - A reply to a command.
  - Uses `cid = id` of the original command.
  - **Shares the same `name` as the original command**, differentiated only by `kind`.
  - A `response` may itself be a success or an error, depending on domain design.

- `kind = "event"`  
  - Fire-and-forget notification.
  - Does not expect a response.
  - Common for engine status, state changes, and logging signals.

- `kind = "snapshot"`  
  - Authoritative state projection from Pulse to Aura.
  - Usually contains a model view (tracks, clips, parameters, etc.).
  - Snapshots may be full or partial; semantics are defined per domain.

- `kind = "error"`  
  - An error response or standalone error event.
  - Contains an `error` block with details.
  - May or may not correlate to a command (via `cid`).

---

## 5. Ordering and Reliability

### 5.1 Per-Connection Ordering

Within a single logical connection:

- Messages are assumed to be received in the order they were sent.
- The envelope does not attempt to impose ordering across multiple connections.

Domains that require stronger guarantees (e.g. monotonically increasing
parameter updates) should specify:

- sequence numbers in their payloads, or
- explicit ordering rules in the domain spec.

### 5.2 Delivery Semantics

For most control and editing flows:

- Delivery is **at-most-once**.  
  Callers may choose to retry commands if they do not receive responses,
  but the envelope layer does not manage retries.

For high-rate streams (e.g. gestures, metering):

- Delivery is **best-effort**.
- Missing frames are tolerated.
- No retries are attempted at the envelope layer.

---

## 6. Binary Fast-Path Frames

For high-rate streams (gesture/control streams, metering, and future
waveform/spectral previews), JSON is not appropriate due to overhead.

Binary frames:

- minimise per-sample or per-bucket overhead,
- are associated with logical streams (identified by `streamId`),
- are usually established and torn down via JSON commands.

### 6.1 Binary Frame Header (Conceptual)

Exact byte layout is implementation-specific, but the conceptual header fields
are:

- `v` – frame version.
- `kind` – e.g. `"gesture"` or `"meter"`.
- `streamId` – identifies the logical stream.
- `sequence` – monotonically increasing sequence per `streamId`.
- `ts` or `timeOffset` – timestamp or offset relative to stream start.

Binary frames:

- may be carried on the same transport as JSON messages, but logically
  multiplexed,
- are not wrapped in JSON envelopes per frame,
- rely on prior JSON-level negotiation to define the meaning of `streamId`.

### 6.2 Stream Setup via JSON

Before any binary frames are sent, the stream is opened using a JSON command.
Example for a gesture stream:

```jsonc
{
  "v": 1,
  "id": "msg-123",
  "cid": null,
  "ts": "2025-11-17T22:34:00.000Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "gesture",
  "kind": "command",
  "name": "open",
  "priority": "realtime",
  "payload": {
    "streamId": "gstrm-abc",
    "parameterId": "track.5.channel.proc.3.cutoff",
    "rateHintHz": 250
  },
  "error": null
}
```

Signal replies with a response/event (domain: `gesture`, name: `opened`). Afterwards,
binary frames using `streamId = "gstrm-abc"` carry the actual gesture data.

Closing a stream is symmetrical, via a command (domain: `gesture`, name: `close`) and
corresponding event/response (domain: `gesture`, name: `closed`).

Metering streams may follow an analogous pattern (`meterStream.open`, etc.).

---

## 7. Error Model

Errors are expressed using standard envelopes with `kind = "error"`.

For example, a response to a failing command:

```jsonc
{
  "v": 1,
  "id": "msg-456",
  "cid": "msg-123",
  "ts": "2025-11-17T22:34:05.000Z",
  "origin": "pulse",
  "target": "aura",
  "domain": "project",
  "kind": "error",
  "name": "open",
  "priority": "high",
  "payload": {},
  "error": {
    "code": "project.notFound",
    "message": "Project file could not be found.",
    "details": {
      "path": "/path/to/file"
    }
  }
}
```

Notes:

- `name` is typically the original command name, so that the error can be
  easily mapped to the request.
- `cid` is the `id` of the original command.
- `error.code` is machine-friendly; `error.message` is suitable for UI/logging.
- `error.details` is optional and domain-specific.

Standalone engine errors (e.g. plugin crashes) may be emitted as events:

```jsonc
{
  "v": 1,
  "id": "msg-789",
  "cid": null,
  "ts": "2025-11-17T22:34:10.000Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "engine",
  "kind": "error",
  "name": "pluginCrashed",
  "priority": "realtime",
  "payload": {
    "nodeId": "track.3.channel.node.2",
    "pluginIdentity": {
      "vendor": "ExampleVendor",
      "name": "ExamplePlugin",
      "version": "1.2.3"
    }
  },
  "error": {
    "code": "engine.pluginCrashed",
    "message": "Plugin crashed during processing.",
    "details": {}
  }
}
```

---

## 8. Versioning Strategy

Versioning operates at two levels:

1. **Envelope version (`v`)**  
   - Controls top-level envelope fields (`id`, `cid`, `ts`, `origin`, `target`,
     `domain`, `kind`, `name`, `priority`, `payload`, `error`).
   - Incremented only when we introduce incompatible changes to the envelope
     shape.

2. **Domain schema versions**  
   - Each domain (e.g. Project, Transport, Track) defines its own payload
     schemas.
   - New fields in payloads should be:

     - optional,  
     - safe to ignore by older peers.

Forwards-compatibility rules:

- New envelope fields must:

  - be optional,  
  - have well-defined defaults when missing.

- Older peers must tolerate unknown fields without failing.

---

## 9. Security and Safety Notes

Detailed security design is out of scope for this document, but the envelope
should be used with the following principles:

- **Strict validation** on receipt:

  - `origin`, `target`, `domain`, `kind`, `name` must be verified against
    known sets,
  - malformed envelopes should be rejected or logged as protocol errors.

- **No JSON parsing on real-time audio threads**:

  - In Signal, all JSON parsing must happen on non-real-time threads.
  - Binary frames should be handled by IO threads feeding lock-free queues
    into the audio engine.

- **Pulse is the policy gate**:

  - Aura cannot directly target Signal.
  - Composer cannot directly talk to Signal or Aura.
  - Pulse enforces routing and filters/normalises inputs.

Transport-level security (authentication, encryption, etc.) will be defined
at deployment/environment level, not in this envelope spec.

---

## 10. Examples

### 10.1 Aura → Pulse: User Pressed Play

```jsonc
{
  "v": 1,
  "id": "msg-1001",
  "cid": null,
  "ts": "2025-11-17T22:34:00.123Z",
  "origin": "aura",
  "target": "pulse",
  "domain": "transport",
  "kind": "command",
  "name": "play",
  "priority": "high",
  "payload": {
    "fromPosition": null
  },
  "error": null
}
```

Pulse will respond by:

- updating internal transport state,
- sending appropriate commands to Signal,
- emitting `transportStateChanged` events and/or snapshots to Aura.

### 10.2 Pulse → Signal: Cohort Graph Update

```jsonc
{
  "v": 1,
  "id": "msg-2001",
  "cid": null,
  "ts": "2025-11-17T22:34:00.456Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "cohort",
  "kind": "command",
  "name": "assign",
  "priority": "high",
  "payload": {
    "graphVersion": 12,
    "assignments": [
      { "nodeId": "track.3.channel.node.1", "cohort": "live" },
      { "nodeId": "track.4.channel.node.2", "cohort": "anticipative" }
    ]
  },
  "error": null
}
```

Signal applies the new cohort assignments outside the audio thread and adjusts
its live/anticipative execution plans accordingly.

### 10.3 Pulse → Aura: Project Snapshot

```jsonc
{
  "v": 1,
  "id": "msg-3001",
  "cid": null,
  "ts": "2025-11-17T22:34:01.000Z",
  "origin": "pulse",
  "target": "aura",
  "domain": "project",
  "kind": "snapshot",
  "name": "snapshot",
  "priority": "normal",
  "payload": {
    "projectId": "proj-abc",
    "version": 5,
    "tracks": [],
    "clips": [],
    "tempoMaps": []
  },
  "error": null
}
```

Aura treats this as the authoritative view of the project model (or
view-specific subset), discarding any conflicting local assumptions.

