# Pulse Client Domain Specification

This document defines the **Client** domain of the Pulse IPC protocol.

The Client domain is responsible for managing **Aura client sessions**:
registration, capabilities, heartbeats, event subscriptions and orderly
shutdown. It ensures Pulse always has a clear view of who is connected, what
they support, and how to shape event delivery.

The Client domain **does not** carry project or engine semantics; it purely
describes the relationship between a connected UI client (Aura) and Pulse.

---

## Contents

- [1. Overview](#1-overview)
  - [1.1 Goals](#11-goals)
  - [1.2 Non-goals](#12-non-goals)
  - [1.3 Identity Model](#13-identity-model)
- [2. Commands (Aura → Pulse)](#2-commands-aura--pulse)
  - [2.1 Session Lifecycle](#21-session-lifecycle)
  - [2.2 Heartbeats](#22-heartbeats)
  - [2.3 Event Subscriptions](#23-event-subscriptions)
  - [2.4 Capabilities and Preferences](#24-capabilities-and-preferences)
- [3. Events (Pulse → Aura)](#3-events-pulse--aura)
  - [3.1 Session Lifecycle Events](#31-session-lifecycle-events)
  - [3.2 Heartbeat and Timeout Events](#32-heartbeat-and-timeout-events)
  - [3.3 Subscription and Capability Events](#33-subscription-and-capability-events)
- [4. Error Handling](#4-error-handling)
- [5. Notes and Constraints](#5-notes-and-constraints)

---

## 1. Overview

The Client domain defines how an Aura instance:

- announces itself to Pulse,
- negotiates a **client/session identifier**,
- declares capabilities and feature flags,
- specifies which events it wants to receive,
- proves it is still alive (heartbeats),
- and shuts down cleanly.

In future, this same domain can be reused for **headless clients** (e.g. test
harnesses) that want to control Pulse without a full UI.

### 1.1 Goals

- Provide a **clear handshake** between Aura and Pulse.
- Allow Pulse to **differentiate clients** (e.g. multiple Auras, tools).
- Support **event subscription** so Aura can control bandwidth and noise.
- Provide **heartbeats** and **timeout detection** without mixing into project
  or engine domains.
- Be safe to extend without breaking core IPC semantics.

### 1.2 Non-goals

- The Client domain does **not** carry project/track/clip/node operations.
- The Client domain does **not** expose debug or introspection features (those
  live in the Debug domain).
- The Client domain does **not** manage authentication or security policies
  (out of scope for initial implementation).

### 1.3 Identity Model

The Client domain uses the following identifiers:

- `clientId` — stable logical identity for the **client program**  
  (e.g. `"aura"`, `"loophole-cli"`). This is provided by the client in registration and echoed back by Pulse. Pulse may use `clientId` for operations such as determining managed status (`isManaged`). This value may be reused across sessions.
- `sessionId` — unique identifier for a **connection/session**  
  (e.g. `"session:8f2c9d..."`). Pulse is authoritative for this value.
- `instanceId` — unique identifier provided by the client describing the specific running instance  
  (e.g. a UUID generated per client process). This is **optional** and purely advisory. Helps clients recognise "same Pulse instance" or track their own instances. Not required for Pulse to operate.

Pulse echoes back `clientId` and `instanceId` in responses to confirm the effective identity for the session.

---

## 2. Commands (Aura → Pulse)

All commands in this domain use:

- `domain: "client"`
- `type: "client.<action>"`

### 2.1 Session Lifecycle

#### `client.register`

Announce a new client connection and request a session.

Payload:

```json
{
  "clientName": "Loophole Aura",
  "clientVersion": "0.1.0-dev",
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "capabilities": {
    "supportsRichSnapshots": true,
    "supportsPartialSnapshots": true,
    "supportsHighFrequencyEvents": true,
    "supportsScriptingPanels": false
  }
}
```

Fields:

- `clientId` (string) — stable logical identity for the client (e.g. `"aura"`, `"loophole-cli"`). Pulse may use this to determine managed status if it was launched with a matching `--client-id`.
- `instanceId` (string | null, optional) — unique identifier for this running instance of the client (e.g. a UUID). Helps clients recognise "same Pulse instance" or track their own instances. Not required for Pulse to operate.

Behaviour:

- Pulse accepts the provided `clientId` and echoes it back in responses.
- Pulse creates a new `sessionId` bound to the underlying connection.
- Pulse records advertised capabilities.
- Pulse determines `isManaged` based on whether the client's `clientId` matches the `--client-id` it was launched with.
- On success, Pulse emits `client.welcome` and `client.registered` events.

#### `client.unregister`

Indicate that the client intends to close the session cleanly.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d..."
}
```

Behaviour:

- Pulse marks the session as closing.
- Pulse stops sending events to this session.
- Pulse may emit `client.unregistered`.
- The underlying transport may then be closed by either side.

If the TCP/WebSocket connection is dropped without `client.unregister`, Pulse
will infer disconnection based on transport events and heartbeat timeouts.

### 2.2 Heartbeats

#### `client.heartbeat`

Periodic liveness signal from Aura.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "seq": 42
}
```

Behaviour:

- Pulse updates last-seen time for the session.
- Pulse may respond with a short `client.heartbeatAck` response **or** may only
  emit events when there is a change (implementation detail).
- Missing heartbeats beyond a configured timeout window result in a
  `client.timedOut` event and eventual session teardown.

Heartbeat cadence is negotiated by configuration (see `client.updatePreferences`).

### 2.3 Event Subscriptions

#### `client.subscribeEvents`

Declare which domains and/or specific event types Aura wants to receive.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "subscriptions": [
    {
      "domain": "project",
      "events": ["project.loaded", "project.snapshot"]
    },
    {
      "domain": "transport"
    },
    {
      "domain": "metering",
      "events": ["metering.levels"],
      "maxFrequencyHz": 30
    }
  ]
}
```

Behaviour:

- Pulse stores subscription preferences with the session.
- Pulse filters outgoing events accordingly.
- If `events` is omitted, all events in the domain are delivered.
- Some high-frequency domains (e.g. `metering`) may honour `maxFrequencyHz`.

#### `client.unsubscribeEvents`

Remove one or more subscriptions.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "domains": ["metering", "diagnostics"]
}
```

Behaviour:

- Pulse removes matching subscriptions for those domains.
- If `domains` is omitted, all subscriptions for the session are cleared.

### 2.4 Capabilities and Preferences

#### `client.updateCapabilities`

Update advertised capabilities at runtime.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "capabilities": {
    "supportsHighFrequencyEvents": false,
    "supportsPluginUIPinning": true
  }
}
```

Behaviour:

- Pulse merges the new capability set into the existing record.
- Pulse may adjust event frequencies or feature usage based on capabilities.

#### `client.updatePreferences`

Update client-specific preferences affecting delivery behaviour.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "preferences": {
    "heartbeatIntervalMs": 5000,
    "maxEventBatchSize": 128
  }
}
```

Behaviour:

- Pulse updates internal delivery settings for this session.
- Invalid or unsupported preference keys are ignored (or produce
  `client.invalidPreferences` errors).

---

## 3. Events (Pulse → Aura)

Events use:

- `domain: "client"`
- `type: "client.<eventName>"`

### 3.1 Session Lifecycle Events

#### `client.welcome`

Emitted after successful `client.register`, providing initial session information including project state if available.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "isManaged": true,
  "pulseVersion": "0.1.0",
  "hasProject": false,
  "projectId": null,
  "projectName": null
}
```

Fields:

- `clientId` (string) — echo of the effective client ID for this session (as provided in `client.register`).
- `instanceId` (string | null) — echo of the instance ID for this session (if provided in `client.register`).
- `isManaged` (boolean) — indicates whether this Pulse instance considers itself to be managed by this client. True when the client's `clientId` matches the `--client-id` Pulse was launched with; false otherwise.
- `pulseVersion` (string) — version of the Pulse server.
- `hasProject` (boolean) — whether a project is currently loaded.
- `projectId` (string | null) — ID of the current project (if any).
- `projectName` (string | null) — name of the current project (if any).

#### `client.registered`

Emitted after successful `client.register`, confirming registration and echoing back client identity.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "isManaged": true
}
```

Fields:

- `clientId` (string) — echo of the effective client ID for this session (as provided in `client.register`).
- `instanceId` (string | null) — echo of the instance ID for this session (if provided in `client.register`).
- `isManaged` (boolean) — indicates whether this Pulse instance considers itself to be managed by this client. True when the client's `clientId` matches the `--client-id` Pulse was launched with; false otherwise.

#### `client.unregistered`

Emitted when a session is closed cleanly (via `client.unregister`) or after a
detected timeout / transport tear-down.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "reason": "clientRequested"  // or "transportClosed", "timeout"
}
```

#### `client.reconnected` (optional / future)

If Pulse supports session resumption, this event signals that a new underlying
connection has been bound to an existing logical session.

Payload:

```json
{
  "clientId": "aura",
  "oldSessionId": "session:old...",
  "newSessionId": "session:new..."
}
```

### 3.2 Heartbeat and Timeout Events

#### `client.heartbeatRequired`

Advisory event letting Aura know it should start sending heartbeats.

Payload:

```json
{
  "intervalMs": 5000,
  "graceMs": 15000
}
```

#### `client.timedOut`

Emitted when a client has not sent heartbeats within the configured grace
period.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "lastHeartbeatAt": "2025-11-19T12:05:00Z"
}
```

Pulse may follow this with `client.unregistered` once cleanup is complete.

### 3.3 Subscription and Capability Events

#### `client.subscriptionsUpdated`

Emitted when subscriptions are successfully changed.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "subscriptions": [
    { "domain": "project" },
    { "domain": "transport" },
    { "domain": "metering", "maxFrequencyHz": 30 }
  ]
}
```

#### `client.capabilitiesUpdated`

Emitted when Pulse has applied new capability information.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "capabilities": {
    "supportsHighFrequencyEvents": false,
    "supportsPluginUIPinning": true
  }
}
```

---

## 4. Error Handling

All errors follow the envelope error schema (`details` field for structured
data).

Examples:

### Invalid Session

```json
{
  "error": {
    "code": "invalidSession",
    "message": "No active session with ID session:deadbeef.",
    "domain": "client",
    "details": {
      "sessionId": "session:deadbeef"
    }
  }
}
```

### Unsupported Capability

```json
{
  "error": {
    "code": "unsupportedCapability",
    "message": "Capability supportsPluginUIPinning is not recognised.",
    "domain": "client",
    "details": {
      "capability": "supportsPluginUIPinning"
    }
  }
}
```

### Subscription Error

```json
{
  "error": {
    "code": "invalidSubscription",
    "message": "Domain metering does not support subscription at this granularity.",
    "domain": "client",
    "details": {
      "domain": "metering",
      "maxFrequencyHz": 1000
    }
  }
}
```

---

## 5. Notes and Constraints

- The Client domain is **infrastructure**, not user-facing modelling.
- A session is tied to a **single transport connection**; reconnect/resume
  behaviour is a future extension.
- Subscriptions are **best-effort filters**, not strict guarantees. High
  priority events may bypass some filters if necessary (e.g. fatal errors).
- Heartbeats and timeouts must be implemented in a way that **never** touches
  real-time Signal processing.
- Debug/diagnostics features should use the **Debug** domain, not the Client
  domain.
- Pulse's management relationship is described via `clientId` and `isManaged`.
  The `instanceId` is auxiliary identity metadata and does not affect managed
  status determination.
