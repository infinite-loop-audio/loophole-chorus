# IPC Client Domain Identity Documentation Update

**Date:** 2025-11-19  
**Implementation:** Update Chorus IPC client domain specification for new client identity model  
**Status:** ✅ Complete

## Summary

Updated the IPC specification for the Client domain to reflect the new client identity model, replacing `proposedClientId` with `clientId`, clarifying `instanceId` semantics, and documenting `isManaged` in responses. Removed all manager-related terminology in favour of the generic client identity approach.

---

## Files Modified

- `docs/specs/ipc/pulse/client.md` — Complete update to client identity model

---

## `client.register` Command Schema

### Previous Schema

```json
{
  "clientName": "Loophole Aura",
  "clientVersion": "0.1.0-dev",
  "proposedClientId": "aura:desktop",
  "instanceId": "aura:studio-mac",
  "capabilities": { ... }
}
```

### Updated Schema

```json
{
  "clientName": "Loophole Aura",
  "clientVersion": "0.1.0-dev",
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "capabilities": { ... }
}
```

### Changes

- **Renamed:** `proposedClientId` → `clientId`
- **Clarified:** `clientId` is now a stable logical identity (e.g. `"aura"`, `"loophole-cli"`) rather than a compound identifier
- **Updated:** `instanceId` now uses UUID format and is clarified as optional metadata

### Field Semantics

- **`clientId` (string)** — stable logical identity for the client (e.g. `"aura"`, `"loophole-cli"`). Pulse may use this to determine managed status if it was launched with a matching `--client-id`.
- **`instanceId` (string | null, optional)** — unique identifier for this running instance of the client (e.g. a UUID). Helps clients recognise "same Pulse instance" or track their own instances. Not required for Pulse to operate.

### Behaviour

- Pulse accepts the provided `clientId` and echoes it back in responses
- Pulse determines `isManaged` based on whether the client's `clientId` matches the `--client-id` it was launched with
- On success, Pulse emits `client.welcome` and `client.registered` events

---

## Identity Model Section

### Updated Semantics

The Identity Model section (1.3) now clearly defines:

- **`clientId`** — stable logical identity for the client program (e.g. `"aura"`, `"loophole-cli"`). Provided by the client in registration and echoed back by Pulse. Pulse may use `clientId` for operations such as determining managed status (`isManaged`). This value may be reused across sessions.
- **`sessionId`** — unique identifier for a connection/session. Pulse is authoritative for this value.
- **`instanceId`** — unique identifier provided by the client describing the specific running instance (e.g. a UUID generated per client process). This is **optional** and purely advisory. Helps clients recognise "same Pulse instance" or track their own instances. Not required for Pulse to operate.

**Key point:** Pulse echoes back `clientId` and `instanceId` in responses to confirm the effective identity for the session.

---

## Client Snapshot / Response Schemas

### `client.welcome` Event

Added new event documentation (previously only `client.registered` was documented):

**Payload:**

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

**Fields:**

- `clientId` (string) — echo of the effective client ID for this session (as provided in `client.register`)
- `instanceId` (string | null) — echo of the instance ID for this session (if provided in `client.register`)
- `isManaged` (boolean) — indicates whether this Pulse instance considers itself to be managed by this client. True when the client's `clientId` matches the `--client-id` Pulse was launched with; false otherwise.
- `pulseVersion` (string) — version of the Pulse server
- `hasProject` (boolean) — whether a project is currently loaded
- `projectId` (string | null) — ID of the current project (if any)
- `projectName` (string | null) — name of the current project (if any)

### `client.registered` Event

**Previous Payload:**

```json
{
  "clientId": "aura:desktop",
  "sessionId": "session:8f2c9d...",
  "assignedAt": "2025-11-19T12:00:00Z",
  "capabilities": { ... }
}
```

**Updated Payload:**

```json
{
  "clientId": "aura",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "isManaged": true
}
```

**Changes:**

- Added `instanceId` field (echo of registration value)
- Added `isManaged` field (boolean indicating managed status)
- Removed `sessionId`, `assignedAt`, and `capabilities` (these may be available elsewhere or were not consistently present)
- Updated `clientId` format to simple identifier (`"aura"` instead of `"aura:desktop"`)

**Fields:**

- `clientId` (string) — echo of the effective client ID for this session (as provided in `client.register`)
- `instanceId` (string | null) — echo of the instance ID for this session (if provided in `client.register`)
- `isManaged` (boolean) — indicates whether this Pulse instance considers itself to be managed by this client. True when the client's `clientId` matches the `--client-id` Pulse was launched with; false otherwise.

---

## `isManaged` Semantics

The `isManaged` field is now explicitly documented in both `client.welcome` and `client.registered` events:

- **Purpose:** Indicates whether this Pulse instance considers itself to be managed by this client
- **Determination:** True when the client's `clientId` matches the `--client-id` Pulse was launched with; false otherwise
- **Usage:** Helps clients understand their relationship with Pulse (e.g. whether they can request shutdown, whether they spawned Pulse, etc.)

**Important:** `instanceId` is **never** used to determine `isManaged`. Only `clientId` is considered.

---

## Removed Manager References

No manager-related structures or fields were found in the client domain spec (they may have been in earlier drafts that were already removed). The spec now consistently uses:

- `clientId` — for client identity
- `instanceId` — for instance metadata
- `isManaged` — for management relationship

All example payloads throughout the document were updated to use `"aura"` instead of `"aura:desktop"` for consistency with the new identity model.

---

## Notes and Constraints Section

Added explicit note:

> Pulse's management relationship is described via `clientId` and `isManaged`. The `instanceId` is auxiliary identity metadata and does not affect managed status determination.

This clarifies that:
- Management is determined solely by `clientId` matching
- `instanceId` is purely informational
- No nested manager structures exist

---

## Consistency Checks

- ✅ No references to `proposedClientId` remain
- ✅ No references to nested `manager` structures or `managerName`/`managerId` fields
- ✅ All example payloads use consistent `clientId` format (`"aura"` instead of `"aura:desktop"`)
- ✅ Client identity is consistently described in terms of `clientId`, `instanceId`, and `isManaged`
- ✅ The spec matches current Pulse/Aura implementation direction
- ✅ No environment variable references (e.g. `PULSE_MANAGER`) appear in the IPC spec (these belong in implementation docs, not IPC contracts)

---

## Index and Reference Files

The client domain is an infrastructure domain and is not explicitly listed in the IPC overview's domain taxonomy (section 5), which focuses on user-facing domains. This is appropriate — the client domain handles connection lifecycle, not project/engine semantics.

No other index or reference files were found that specifically mention client identity semantics requiring updates.

---

## Summary of Changes

### Command Schema (`client.register`)

- `proposedClientId` → `clientId`
- Clarified `clientId` as stable logical identity
- Updated `instanceId` to UUID format with clarified semantics

### Event Schemas

- Added `client.welcome` event documentation
- Updated `client.registered` to include `clientId`, `instanceId`, and `isManaged`
- Removed fields that were not consistently present

### Identity Model

- Updated semantics to reflect client-provided `clientId` (not Pulse-assigned)
- Clarified `instanceId` as optional, advisory metadata
- Documented that Pulse echoes back identity values

### Management Relationship

- Documented `isManaged` boolean field
- Explicitly stated that only `clientId` affects managed status
- Clarified that `instanceId` is not used for management decisions

The specification now accurately reflects the implementation in Pulse and Aura, where clients provide their identity during registration and Pulse determines managed status based on CLI arguments.

