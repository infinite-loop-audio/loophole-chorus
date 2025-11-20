# Pulse Debug Domain Specification

This document defines the **Debug** domain of the Pulse IPC protocol.

The Debug domain provides **introspection and development tools** for Loophole.
It allows developers and advanced users to inspect internal state, adjust log
verbosity, and run controlled test actions without affecting core IPC semantics
for normal clients.

The Debug domain is considered **optional** and is expected to be:

- gated behind a configuration flag or build profile,
- disabled (or heavily restricted) in production builds,
- never required for normal operation.

---

## Contents

- [1. Overview](#1-overview)
  - [1.1 Goals](#11-goals)
  - [1.2 Non-goals](#12-non-goals)
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)
  - [2.1 Logging and Diagnostics Control](#21-logging-and-diagnostics-control)
  - [2.2 Introspection and State Inspection](#22-introspection-and-state-inspection)
  - [2.3 Test and Simulation Utilities](#23-test-and-simulation-utilities)
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)
  - [3.1 Log and Diagnostic Events](#31-log-and-diagnostic-events)
  - [3.2 Introspection Events](#32-introspection-events)
- [4. Error Handling](#4-error-handling)
- [5. Safety and Constraints](#5-safety-and-constraints)

---

## 1. Overview

The Debug domain is a **side-channel** used during development, testing and
advanced troubleshooting. It sits beside, not inside, the core model and engine
behaviour.

It is intentionally narrow in scope:

- control and query logging and diagnostics,
- provide structured snapshots for debugging,
- simulate limited conditions to exercise error paths,
- never mutate the project model in ways that bypass normal domain commands.

### 1.1 Goals

- Provide **structured access** to internal state for debugging.
- Allow dynamic adjustment of **log verbosity** and diagnostic output.
- Offer a minimal set of **test hooks** to validate IPC paths and error
  handling.
- Remain **safe to remove** in minimal builds.

### 1.2 Non-goals

- The Debug domain is **not** a replacement for proper architecture docs.
- It is **not** a backdoor for editing projects; editing must go through
  normal domains (project, track, clip, etc.).
- It is **not** a performance profiler; detailed performance metrics live in
  diagnostics-specific subsystems (see diagnostics architecture).

---

## 2. Commands (Aura → Pulse)

All commands in this domain use:

- `domain: "debug"`
- `name: "<action>"`

### 2.1 Logging and Diagnostics Control

#### `debug.setLogLevel`

Adjust the verbosity of Pulse logs for the current process.

Payload:

```json
{
  "level": "info"  // "trace" | "debug" | "info" | "warn" | "error" | "off"
}
```

Behaviour:

- Pulse updates its logging configuration accordingly.
- May not be supported in all build profiles; if unsupported, returns an
  error.

#### `debug.enableDomainLogging`

Enable or disable logging for specific IPC domains (for debugging noisy
interactions).

Payload:

```json
{
  "domains": ["project", "track", "transport"],
  "enabled": true
}
```

Behaviour:

- Pulse updates internal tracing filters for the specified domains.
- Exact implementation is left to Pulse (may map to tracing spans, tag-based
  logs, etc.).

### 2.2 Introspection and State Inspection

#### `debug.getStateSummary`

Request a coarse-grained summary of current Pulse state for debugging.

Payload:

```json
{
  "includeCounts": true,
  "includeProjectInfo": true
}
```

Typical response payload:

```json
{
  "project": {
    "loaded": true,
    "projectId": "project:abcd",
    "name": "My Song",
    "tracks": 24,
    "clips": 128
  },
  "engine": {
    "connected": true,
    "graphVersion": 7
  },
  "clients": {
    "activeSessions": 1
  }
}
```

This command is intended for **debug panels** and should not be used in normal
runtime flows.

#### `debug.getDomainState`

Request a domain-specific state snapshot for debugging.

Payload:

```json
{
  "domain": "track",
  "options": {
    "includeMuted": true
  }
}
```

Behaviour:

- Pulse returns a domain-specific snapshot in the response payload.
- Structure is **not guaranteed stable** between versions; this is a debug
  facility, not part of the core schema.

Example response payload:

```json
{
  "tracks": [
    {
      "trackId": "track:1",
      "name": "Kick",
      "muted": false,
      "armed": true
    }
  ]
}
```

### 2.3 Test and Simulation Utilities

These commands are **optional** and may not be available in all builds. They
are intended for testing IPC paths, error handling and diagnostics UI.

#### `debug.echo`

Round-trip echo of an arbitrary JSON payload, used to validate IPC wiring.

Payload:

```json
{
  "payload": {
    "message": "hello",
    "value": 123
  }
}
```

Response payload:

```json
{
  "payload": {
    "message": "hello",
    "value": 123
  }
}
```

#### `debug.simulateError`

Ask Pulse to emit a structured error event, for testing error handling in Aura.

Payload:

```json
{
  "domain": "project",
  "code": "simulatedFailure",
  "message": "This is a simulated error for testing.",
  "details": {
    "note": "no real project was harmed"
  }
}
```

Behaviour:

- Pulse emits a domain-appropriate error event (or a generic one) to the
  calling client.
- Pulse does **not** mutate real project state.

---

## 3. Events (Pulse → Aura)

Events use:

- `domain: "debug"`
- `name: "<eventName>"`

### 3.1 Log and Diagnostic Events

#### `debug.log`

Structured log event forwarded to Aura, typically for a debug console.

Payload:

```json
{
  "level": "info",      // "trace" | "debug" | "info" | "warn" | "error"
  "timestamp": "2025-11-19T12:34:56Z",
  "category": "ipc.transport",
  "message": "Received envelope from client.",
  "details": {
    "origin": "aura",
    "domain": "project",
    "name": "open"
  }
}
```

#### `debug.diagnostic`

General-purpose diagnostic event for ad-hoc debug data.

Payload:

```json
{
  "kind": "sessionState",
  "timestamp": "2025-11-19T12:35:01Z",
  "details": {
    "activeSessions": 1,
    "openProjects": 1
  }
}
```

### 3.2 Introspection Events

Some introspection commands may emit follow-up events (in addition to
responses) where streaming is more convenient than a single large response.

#### `debug.stateChunk`

Optional event used when a state snapshot is emitted in chunks.

Payload:

```json
{
  "requestId": "req:1234",
  "chunkIndex": 0,
  "chunkCount": 2,
  "data": {
    "tracks": [
      { "trackId": "track:1", "name": "Kick" }
    ]
  }
}
```

---

## 4. Error Handling

As with all domains, errors use the envelope `error` object with `details`
for structured information.

Examples:

### Debug Feature Disabled

```json
{
  "error": {
    "code": "debugDisabled",
    "message": "Debug domain is disabled in this build.",
    "domain": "debug",
    "details": {}
  }
}
```

### Unsupported Command

```json
{
  "error": {
    "code": "unsupportedCommand",
    "message": "Command debug.simulateError is not available.",
    "domain": "debug",
    "details": {
      "command": "debug.simulateError"
    }
  }
}
```

### Invalid Introspection Request

```json
{
  "error": {
    "code": "invalidDomain",
    "message": "Domain 'foobar' cannot be inspected.",
    "domain": "debug",
    "details": {
      "requestedDomain": "foobar"
    }
  }
}
```

---

## 5. Safety and Constraints

- The Debug domain must be **clearly separated** from core editing domains.
- It MUST NOT:
  - bypass normal validation rules,
  - mutate project state in ways that cannot be reproduced via normal commands,
  - touch Signal’s real-time threads directly.
- It SHOULD:
  - be easy to disable for production builds,
  - be guarded by configuration (and potentially authentication) in
    multi-user/remote variants,
  - treat all returned structures as **unstable** across versions.

In short: Debug is a **toolbox for development and diagnostics**, not a
user-facing editing API.
