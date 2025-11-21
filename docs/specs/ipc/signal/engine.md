# Signal IPC – Engine Domain

This document defines the **Engine** IPC domain between Pulse and Signal.

The Engine domain is responsible for:

- high-level lifecycle control of the audio engine process,
- configuration of global engine parameters (sample rate, block size, modes),
- reporting engine state and fatal/non-fatal errors,
- providing a capability/handshake layer so Pulse knows what Signal can do.

The Engine domain is intentionally **small and low-frequency**. It does not
manage Tracks, Nodes, Media, or Transport directly; those are covered by other
Signal IPC domains.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Engine domain:

- starts and stops the DSP engine safely,
- applies global configuration:
  - nominal sample rate,
  - nominal block size / buffer size,
  - processing mode flags (e.g. real-time + anticipative rendering),
  - high-level resource limits,
- reports:
  - engine lifecycle state (starting, running, stopping, stopped),
  - safe-mode status,
  - critical errors that require user intervention.

### 1.2 Non-Goals

The Engine domain does **not**:

- manage the audio/MIDI device selection directly (handled by `signal.hardware`),
- define or modify the graph (Channels/Nodes) (handled by `signal.graph`),
- control transport state (play/stop/seek) (handled by `signal.transport`),
- handle per-node or per-cohort configuration (handled by `signal.graph` and
  `signal.diagnostics`),
- perform media decoding or analysis (`signal.media`),
- open plugin UIs or manage plugins (`signal.plugin`).

---

## 2. Message Flow Overview

- All Engine IPC messages flow between **Pulse ⇄ Signal**.
- Aura never talks to Signal directly; Aura goes via Pulse.
- Engine commands are always **Pulse → Signal**.
- Engine events are always **Signal → Pulse**.

A typical lifecycle:

1. Pulse starts and connects to Signal.
2. Pulse sends `engine.handshake` to discover capabilities and protocol
   versions.
3. Pulse sends `engine.configure` with desired settings.
4. Pulse sends `engine.start` when ready.
5. Signal emits `engine.stateChanged` and `engine.configResolved` as it starts.
6. If a fatal error occurs, Signal emits `engine.fatalError` and may request
   shutdown or safe mode.
7. Pulse may call `engine.stop` or `engine.restart` as needed.

---

## 3. Commands (Pulse → Signal)

### 3.1 `engine.handshake`

Initial handshake after establishing IPC connection.

**Purpose**

- Discover Signal’s supported protocol version, capabilities, and constraints.

**Request**

```json
{
  "command": "engine.handshake",
  "clientVersion": "x.y.z",
  "capabilities": {
    "anticipativePreferred": true,
    "maxSampleRateHint": 192000
  }
}
```

**Response**

```json
{
  "replyTo": "engine.handshake",
  "engineVersion": "x.y.z",
  "protocolVersion": 1,
  "capabilities": {
    "supportsAnticipative": true,
    "supportsRealtimeSafeMode": true,
    "supportsDynamicReconfigure": true,
    "maxChannels": 2048,
    "maxSampleRate": 192000
  }
}
```

Signal may refuse the handshake (e.g. incompatible protocol) via an error
response (see §5).

---

### 3.2 `engine.configure`

Apply or update the **global engine configuration**.

**Purpose**

- Set or change nominal engine-wide parameters that affect DSP and device
  negotiation.

**Request**

```json
{
  "command": "engine.configure",
  "sampleRate": 48000,
  "blockSize": 256,
  "processingMode": {
    "anticipativeEnabled": true,
    "realtimeSafeMode": false
  },
  "limits": {
    "maxGraphLatencyMs": 200,
    "maxBackgroundLoadPercent": 70
  }
}
```

Notes:

- `sampleRate` and `blockSize` are **desired** values. The actual values may
  differ once hardware constraints are applied (reported via a subsequent
  event; see §4.2).
- `processingMode` defines global flags. Detailed cohort-level configuration is
  handled in `signal.graph`.

There is no direct response payload beyond generic `ok`/`error`. If the
configuration produces a new effective configuration (e.g. due to hardware
rounding), Signal will emit `engine.configResolved`.

---

### 3.3 `engine.start`

Request the engine to begin active audio processing.

**Request**

```json
{
  "command": "engine.start"
}
```

Semantics:

- If already running, this command is **idempotent** and should result in no
  state change (but may still emit a no-op `engine.stateChanged` depending on
  implementation).
- Signal should only start if a valid configuration and hardware device
  selection are in place. Otherwise it should respond with an error.

---

### 3.4 `engine.stop`

Request the engine to stop audio processing.

**Request**

```json
{
  "command": "engine.stop",
  "reason": "userRequested" // optional, string
}
```

Semantics:

- Engine transitions to a stopped state, releasing real-time resources but
  keeping configuration in memory where possible.
- If already stopped, the command is idempotent.

---

### 3.5 `engine.restart`

Convenience command for stop → start with optional reconfiguration.

**Request**

```json
{
  "command": "engine.restart",
  "reconfigure": {
    "sampleRate": 44100,
    "blockSize": 512
  }
}
```

Semantics:

- Equivalent to `engine.stop`, then (if `reconfigure` present)
  `engine.configure`, then `engine.start`.
- Used when device or sample rate changes require a full restart.

---

### 3.6 `engine.setSafeMode`

Toggle or request “safe mode” operation.

Safe mode is a degraded but robust configuration that might, for example:

- disable anticipative processing,
- use larger internal buffers,
- restrict plugin/node complexity.

**Request**

```json
{
  "command": "engine.setSafeMode",
  "enabled": true,
  "reason": "repeatedDropouts"
}
```

Semantics:

- The specific safe-mode behaviour is implementation-defined, but MUST be
  observable via `engine.stateChanged` and/or `engine.configResolved`.
- Pulse may use this to recover from persistent DSP instability.

---

## 4. Events (Signal → Pulse)

### 4.1 `engine.stateChanged`

Reports high-level lifecycle and safety state.

**Payload**

```json
{
  "event": "engine.stateChanged",
  "state": "running", // "starting" | "running" | "stopping" | "stopped" | "error"
  "safeMode": false,
  "details": {
    "lastError": null
  }
}
```

Notes:

- Emitted on any significant engine lifecycle transition.
- If `state` is `"error"`, additional detail should be provided via
  `engine.fatalError` or `engine.warning`.

---

### 4.2 `engine.configResolved`

Reports the **effective** engine configuration after applying hardware and
internal constraints.

**Payload**

```json
{
  "event": "engine.configResolved",
  "effective": {
    "sampleRate": 47999.8,
    "blockSize": 256,
    "processingMode": {
      "anticipativeEnabled": true,
      "realtimeSafeMode": false
    }
  },
  "hardware": {
    "deviceSampleRate": 48000,
    "deviceBlockSize": 256
  }
}
```

Semantics:

- Emitted after `engine.configure` if the effective configuration differs from
  requested values.
- May also be emitted when device changes occur (coordinated with
  `signal.hardware` events).

---

### 4.3 `engine.warning`

Non-fatal issues that Pulse should surface in Session warnings.

**Payload**

```json
{
  "event": "engine.warning",
  "code": "engineHighLoad", // example
  "message": "Engine DSP load exceeded 90% on several callbacks.",
  "details": {
    "maxLoadPercent": 93.5,
    "durationMs": 2500
  }
}
```

Notes:

- These do not stop the engine.
- Pulse may aggregate or throttle them before showing in Aura.

---

### 4.4 `engine.fatalError`

A serious error requiring user attention and potentially engine shutdown.

**Payload**

```json
{
  "event": "engine.fatalError",
  "code": "deviceLost",
  "message": "The selected audio device became unavailable.",
  "details": {
    "deviceId": "CoreAudio:XYZ",
    "recoverable": true
  }
}
```

Semantics:

- `recoverable` indicates whether Pulse/Signal can attempt an automatic
  recovery (e.g. via `engine.restart` and device reassignment).
- After a fatal error, `engine.stateChanged` with `state = "error"` or
  `state = "stopped"` should follow.

---

### 4.5 `engine.shutdownRequested`

Signal may request a **graceful shutdown** of the engine process, normally due
to unrecoverable internal errors or environment issues.

**Payload**

```json
{
  "event": "engine.shutdownRequested",
  "reason": "internalFailure",
  "message": "Internal assertion failure in DSP engine. Restart is recommended."
}
```

Pulse is responsible for:

- deciding whether to honour the request immediately,
- saving any necessary state,
- informing Aura.

---

## 5. Error Handling

All Engine commands can return an error envelope instead of a normal success
response. The precise envelope shape is defined in the IPC envelope spec; here
we assume a generic pattern:

```json
{
  "error": {
    "code": "invalidConfiguration",
    "message": "Requested sample rate is not supported by any active device.",
    "domain": "engine",
    "details": {
      "requestedSampleRate": 123456
    }
  }
}
```

Typical error codes (non-exhaustive):

- `protocolMismatch`
- `invalidConfiguration`
- `deviceNotReady`
- `engineAlreadyRunning`
- `engineNotRunning`
- `operationNotSupported`

Errors emitted asynchronously are usually surfaced as `engine.warning` or
`engine.fatalError` events, not as direct command responses.

---

## 6. Configuration Application Semantics

The Engine domain does not participate directly in project snapshots, but it
does manage a distinct global **engine configuration state**.

Rules:

- `engine.configure` describes a **desired** configuration.
- `engine.configResolved` describes the **effective** configuration.
- Pulse MUST treat `engine.configResolved` as **authoritative** for any logic
  that depends on actual sample rate or block size.
- When unsure, Pulse must prefer the effective configuration over local
  assumptions.

Engine configuration is orthogonal to project state:

- Projects should not depend on a specific engine instance or device.
- Projects may store **preferences** (desired sample rate) but must remain
  loadable even if the engine configuration differs.

---

## 7. Versioning & Compatibility

The Engine domain is versioned alongside the overall Signal IPC protocol.

- `engine.handshake` advertises `protocolVersion` so Pulse can adapt behaviour.
- Backwards-compatible changes:
  - adding new optional fields to commands or events,
  - adding new non-breaking `processingMode` flags.
- Breaking changes require an increment of `protocolVersion` and conditional
  logic in Pulse.

If Pulse detects an incompatible `protocolVersion`, it should:

- refuse to proceed with normal operation,
- provide a clear error to Aura,
- offer guidance to the user (e.g. “update application or engine”).

---

## 8. Relationship to Other Signal Domains

- **signal.hardware**  
  Responsible for device selection, enumeration and low-level configuration.
  Engine works with hardware to derive effective sample rate and block size.

- **signal.transport**  
  Depends on Engine being in a running state. Transport commands are invalid or
  ignored if the engine is stopped.

- **signal.graph**  
  Graph configuration is only applied meaningfully when the engine is configured;
  some implementations may allow pre-graph setup even when stopped.

- **signal.diagnostics**  
  Provides detailed load and performance metrics that inform Engine-level
  decisions (e.g. enabling safe mode).

The Engine domain is the root of the Signal IPC tree: without a successful
handshake and configuration, other domains cannot operate reliably.
