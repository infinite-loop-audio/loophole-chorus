# Pulse Client Domain Specification

This document defines the **Client** domain of the Pulse IPC protocol.

The Client domain is responsible for managing **Aura client sessions**:
handshake, capabilities, heartbeats, event subscriptions and orderly
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

- announces itself to Pulse via the `hello` handshake,
- negotiates a **client/session identifier**,
- declares capabilities and feature flags,
- specifies which events it wants to receive,
- responds to heartbeat commands from Pulse,
- and shuts down cleanly.

In future, this same domain can be reused for **headless clients** (e.g. test
harnesses) that want to control Pulse without a full UI.

### 1.1 Goals

- Provide a **clear handshake** between Aura and Pulse.
- Allow Pulse to **differentiate clients** (e.g. multiple Auras, tools).
- Support **event subscription** so Aura can control bandwidth and noise.
- Provide **heartbeat commands** (Pulse → Client) and **timeout detection** without mixing into project
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
  (e.g. `"aura"`, `"loophole-cli"`). This is provided by the client in the handshake and echoed back by Pulse. Pulse uses `clientId` to determine manager status (`isManager`). This value may be reused across sessions. A single UI client with a given `clientId` is designated as the "manager"; if a new instance with the same `clientId` connects, it may reclaim management.
- `instanceId` — unique identifier provided by the client describing the specific running instance  
  (e.g. a UUID generated per client process). This is **required** in the handshake and helps clients recognise "same Pulse instance" or track their own instances. Manager status is determined **only by `clientId`**, not `instanceId`.
- `sessionId` — unique identifier for a **connection/session**  
  (e.g. `"session:8f2c9d..."`). Pulse is authoritative for this value.

**Manager Semantics:**

- A single UI client is designated "manager" based solely on matching `clientId`.
- Manager status is determined **only by `clientId`**, not `instanceId`.
- A newly started instance of the same `clientId` may **reclaim management**.
- The `isManager` field in responses indicates whether the current client is the manager.
- `managerClientId` and `managerInstanceId` in responses indicate which client (if any) is currently the manager.

Pulse echoes back `clientId` and `instanceId` in responses to confirm the effective identity for the session.

---

## 2. Commands (Aura → Pulse)

All commands in this domain use:

- `domain: "client"`
- `name: "<action>"`

### 2.1 Session Lifecycle

#### hello (command — domain: client)

Announce a new client connection and request a session. This is the initial handshake command sent by Aura when establishing a connection to Pulse.

**Command Envelope (Aura → Pulse):**

```jsonc
{
  "v": 1,
  "id": "hello-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "aura",
  "target": "pulse",
  "domain": "client",
  "kind": "command",
  "name": "hello",
  "priority": "high",
  "payload": {
    "clientId": "aura",
    "instanceId": "aura:550e8400-e29b-41d4-a716-446655440000",
    "clientName": "Loophole Aura",
    "clientVersion": "0.1.0-dev",
    "capabilities": {
      "supportsRichSnapshots": true,
      "supportsPartialSnapshots": true,
      "supportsHighFrequencyEvents": true,
      "supportsScriptingPanels": false
    }
  },
  "error": null
}
```

**Request Payload:**

```json
{
  "clientId": "aura",
  "instanceId": "aura:550e8400-e29b-41d4-a716-446655440000",
  "clientName": "Loophole Aura",
  "clientVersion": "0.1.0-dev",
  "capabilities": {
    "supportsRichSnapshots": true,
    "supportsPartialSnapshots": true,
    "supportsHighFrequencyEvents": true,
    "supportsScriptingPanels": false
  }
}
```

**Request Fields:**

- `clientId` (string, required) — stable logical identity for the client (e.g. `"aura"`, `"loophole-cli"`). Pulse uses this to determine managed status if it was launched with a matching `--client-id`.
- `instanceId` (string, required) — unique identifier for this running instance of the client (e.g. `"aura:<uuid>"`). Helps clients recognise "same Pulse instance" or track their own instances.
- `clientName` (string, optional) — human-readable name of the client (e.g. `"Loophole Aura"`).
- `clientVersion` (string, optional) — version string of the client (e.g. `"0.1.0-dev"`).
- `capabilities` (object, optional) — map of capability flags indicating what the client supports.

**Response Envelope (Pulse → Aura):**

Pulse responds with a response envelope using:
- `domain: "client"`
- `kind: "response"`
- `name: "hello"` (same name as the command, differentiated by `kind`)
- `cid: <id of the hello command>`

```jsonc
{
  "v": 1,
  "id": "hello-response-456",
  "cid": "hello-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "pulse",
  "target": "aura",
  "domain": "client",
  "kind": "response",
  "name": "hello",
  "priority": "high",
  "payload": {
    "clientId": "aura",
    "instanceId": "aura:550e8400-e29b-41d4-a716-446655440000",
    "serverVersion": "0.1.0-dev",
    "protocolVersion": "1",
    "isManager": true,
    "managerClientId": "aura",
    "managerInstanceId": "aura:550e8400-e29b-41d4-a716-446655440000"
  },
  "error": null
}
```

**Response Payload:**

```json
{
  "clientId": "aura",
  "instanceId": "aura:550e8400-e29b-41d4-a716-446655440000",
  "serverVersion": "0.1.0-dev",
  "protocolVersion": "1",
  "isManager": true,
  "managerClientId": "aura",
  "managerInstanceId": "aura:550e8400-e29b-41d4-a716-446655440000"
}
```

**Response Fields:**

- `clientId` (string) — echo of the effective client ID for this session (as provided in the request).
- `instanceId` (string) — echo of the instance ID for this session (as provided in the request).
- `serverVersion` (string) — version of the Pulse server.
- `protocolVersion` (string) — IPC protocol version supported by Pulse.
- `isManager` (boolean) — indicates whether this client is the manager. True when this client's `clientId` matches the manager's `clientId`; false otherwise. Manager status is determined **only by `clientId`**, not `instanceId`.
- `managerClientId` (string | null) — the `clientId` of the current manager (if any).
- `managerInstanceId` (string | null) — the `instanceId` of the current manager (if any).

**Behaviour:**

- Pulse accepts the provided `clientId` and `instanceId` and echoes them back in the response.
- Pulse determines manager status based solely on `clientId` matching. A newly started instance with the same `clientId` may reclaim management.
- Pulse stores the client identity on the session state.
- Pulse responds with a response envelope (kind="response", name="hello") containing the handshake response payload.
- Pulse may also emit `welcome` and `connected` events after the handshake (these are informational events, not the response to the command).

**Error Handling:**

If the payload is invalid or malformed, Pulse sends an error envelope (kind="error") with an appropriate error code (e.g. `"invalidPayload"`, `"missingField"`).

#### goodbye (command — domain: client)

Indicate that the client intends to close the session cleanly.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "aura:550e8400-e29b-41d4-a716-446655440000"
}
```

Behaviour:

- Pulse marks the session as closing.
- Pulse stops sending events to this session.
- Pulse responds with a response envelope (kind="response", name="goodbye").
- Pulse may emit `disconnected` events.
- The underlying transport may then be closed by either side.

If the TCP/WebSocket connection is dropped without sending `goodbye`, Pulse
will infer disconnection based on transport events and heartbeat timeouts.

### 2.2 Heartbeats

#### heartbeat (command — domain: client)

**Pulse sends** periodic heartbeat commands to all connected UI clients to verify liveness.

**Command (Pulse → Client):**

```jsonc
{
  "v": 1,
  "id": "heartbeat-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "aura",
  "domain": "client",
  "kind": "command",
  "name": "heartbeat",
  "priority": "normal",
  "payload": {}
}
```

**Response (Client → Pulse):**

All UI clients **must reply** with:

```jsonc
{
  "v": 1,
  "id": "heartbeat-response-456",
  "cid": "heartbeat-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "aura",
  "target": "pulse",
  "domain": "client",
  "kind": "response",
  "name": "heartbeat",
  "priority": "normal",
  "payload": {}
}
```

**Behaviour:**

- Pulse sends heartbeat commands at regular intervals to all connected clients.
- Clients **must** respond with a response envelope using the same `name: "heartbeat"` and setting `cid` to the command's `id`.
- Pulse tracks missing responses and emits timeout events when responses are not received within the grace period.
- Missing responses beyond a configured timeout window result in:
  - `client.timedOut` event
  - `client.disconnected` event
  - eventual session teardown

Heartbeat cadence is configured by Pulse and may be adjusted based on connection characteristics.

### 2.3 Event Subscriptions

#### subscribeEvents (command — domain: client)

Declare which domains and/or specific event types Aura wants to receive.

Payload:

```json
{
  "clientId": "aura",
  "sessionId": "session:8f2c9d...",
  "subscriptions": [
    {
      "domain": "project",
      "events": ["loaded", "snapshot"]
    },
    {
      "domain": "transport"
    },
    {
      "domain": "metering",
      "events": ["levels"],
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

#### unsubscribeEvents (command — domain: client)

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

#### updateCapabilities (command — domain: client)

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

#### updatePreferences (command — domain: client)

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
  `invalidPreferences` errors).

---

## 3. Events (Pulse → Aura)

Events use:

- `domain: "client"`
- `name: "<eventName>"`

### 3.1 Session Lifecycle Events

#### welcome (event — domain: client)

Emitted after successful client handshake, providing initial session information including project state if available.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "isManager": true,
  "pulseVersion": "0.1.0",
  "hasProject": false,
  "projectId": null,
  "projectName": null
}
```

Fields:

- `clientId` (string) — echo of the effective client ID for this session (as provided in the hello command).
- `instanceId` (string) — echo of the instance ID for this session (as provided in the hello command).
- `isManager` (boolean) — indicates whether this client is the manager. True when this client's `clientId` matches the manager's `clientId`; false otherwise.
- `pulseVersion` (string) — version of the Pulse server.
- `hasProject` (boolean) — whether a project is currently loaded.
- `projectId` (string | null) — ID of the current project (if any).
- `projectName` (string | null) — name of the current project (if any).

#### connected (event — domain: client)

Emitted after successful client handshake, confirming connection and echoing back client identity.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "isManager": true
}
```

Fields:

- `clientId` (string) — echo of the effective client ID for this session (as provided in the hello command).
- `instanceId` (string) — echo of the instance ID for this session (as provided in the hello command).
- `isManager` (boolean) — indicates whether this client is the manager. True when this client's `clientId` matches the manager's `clientId`; false otherwise.

#### disconnected (event — domain: client)

Emitted when a session is closed cleanly (via `goodbye` command) or after a
detected timeout / transport tear-down.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "reason": "clientRequested"  // or "transportClosed", "timeout"
}
```

#### reconnected (event — domain: client) (optional / future)

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

#### timedOut (event — domain: client)

Emitted when a client has not responded to heartbeat commands within the configured grace
period.

Payload:

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "lastHeartbeatAt": "2025-11-19T12:05:00Z"
}
```

Pulse may follow this with `disconnected` events once cleanup is complete.

### 3.3 Subscription and Capability Events

#### subscriptionsUpdated (event — domain: client)

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

#### capabilitiesUpdated (event — domain: client)

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
- Pulse's management relationship is described via `clientId` and `isManager`.
  Manager status is determined **only by `clientId`**, not `instanceId`.
  The `instanceId` is auxiliary identity metadata used for tracking specific
  client instances but does not affect manager status determination.
- **Commands and responses share the exact same `name`**, differentiated only by `kind`.
- **Events must never reuse a command name**. For example, `client.connected` is an event
  and does not conflict with the `client.hello` command.
