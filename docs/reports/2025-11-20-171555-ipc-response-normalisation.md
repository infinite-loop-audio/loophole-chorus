# IPC Response Normalisation Report

**Date:** 2025-11-20  
**Task:** Align response names and kinds across IPC specifications

## Summary

This report documents the normalisation of IPC response naming and classification across all IPC specifications in the Loophole Chorus repository. The goal was to ensure that:

1. Direct responses to commands use `kind: "response"` with `name` identical to the command's `name`
2. Messages not directly replying to commands are classified as `kind: "event"` instead of `kind: "response"`

## Files Modified

### Core IPC Specifications

1. **`docs/specs/ipc/envelope.md`**
   - Enhanced documentation of `kind: "response"` to explicitly state that responses must share the exact same `name` as their commands
   - Added clarification that only direct replies to commands use `kind: "response"`, and that non-direct replies must use `kind: "event"`
   - Updated naming conventions section to explicitly forbid response name variants like `helloResponse`, `client.helloResponse`, etc.

### Signal Domain Specifications

2. **`docs/specs/ipc/signal/plugin.md`**
   - Converted all command/response pairs from old format (using `replyTo` fields) to proper envelope format
   - Updated 11 command/response pairs:
     - `plugin.listAvailable` → response `name: "listAvailable"`
     - `plugin.rescan` → response `name: "rescan"`
     - `plugin.createInstance` → response `name: "createInstance"`
     - `plugin.destroyInstance` → response `name: "destroyInstance"`
     - `plugin.getInfo` → response `name: "getInfo"`
     - `plugin.getParameters` → response `name: "getParameters"`
     - `plugin.getPrograms` → response `name: "getPrograms"`
     - `plugin.setProgram` → response `name: "setProgram"`
     - `plugin.getState` → response `name: "getState"`
     - `plugin.setState` → response `name: "setState"`
     - `plugin.assessDeterminism` → response `name: "assessDeterminism"`

3. **`docs/specs/ipc/signal/graph.md`**
   - Converted 4 command/response pairs to proper envelope format:
     - `graph.applySnapshot` → response `name: "applySnapshot"`
     - `graph.applyDelta` → response `name: "applyDelta"`
     - `graph.setCohorts` → response `name: "setCohorts"`
     - `graph.querySummary` → response `name: "querySummary"`

4. **`docs/specs/ipc/signal/hardware.md`**
   - Converted 3 command/response pairs to proper envelope format:
     - `hardware.listAudioDevices` → response `name: "listAudioDevices"`
     - `hardware.listMidiDevices` → response `name: "listMidiDevices"`
     - `hardware.getAudioConfig` → response `name: "getAudioConfig"`

### Pulse Domain Specifications

5. **`docs/specs/ipc/pulse/plugin-library.md`**
   - Clarified that events like `pluginLibrary.summary`, `pluginLibrary.chains`, and `pluginLibrary.presets` are events, not direct response envelopes
   - Updated command descriptions to state that Pulse "emits" events rather than "responds with" responses
   - Changed wording from "Response to" to "Emitted in response to" for clarity

6. **`docs/specs/ipc/pulse/hardware-io.md`**
   - Clarified that `hardware.diagnosticsUpdated` is an event, not a direct response envelope
   - Updated wording to explicitly state it's an event emitted in response to `hardware.requestDiagnostics`

7. **`docs/specs/ipc/pulse/engine-diagnostics.md`**
   - Clarified that `engineDiagnostics.snapshotUpdated` is an event, not a direct response envelope
   - Updated wording to explicitly state it's an event emitted in response to `engineDiagnostics.requestSnapshot`

## Key Changes

### Response Name Normalisation

All command/response pairs now follow the rule that responses use the exact same `name` as their commands:

**Before (incorrect):**
- Command: `plugin.listAvailable`
- Response: `replyTo: "plugin.listAvailable"` (old format)

**After (correct):**
- Command: `domain: "plugin", kind: "command", name: "listAvailable"`
- Response: `domain: "plugin", kind: "response", name: "listAvailable"`

### Event Reclassification

Several messages that were described as "responses" but are not direct replies to commands have been clarified as events:

**Examples:**
- `pluginLibrary.summary` — clarified as event (not response to `pluginLibrary.requestSummary`)
- `hardware.diagnosticsUpdated` — clarified as event (not response to `hardware.requestDiagnostics`)
- `engineDiagnostics.snapshotUpdated` — clarified as event (not response to `engineDiagnostics.requestSnapshot`)

## Remaining Work

The following Signal domain files still contain old-style "Response" sections with `replyTo` fields that should be converted to proper envelope format in future work:

- `docs/specs/ipc/signal/plugin-ui.md` (1 response)
- `docs/specs/ipc/signal/gesture.md` (1 response)
- `docs/specs/ipc/signal/engine.md` (1 response)
- `docs/specs/ipc/signal/media.md` (5 responses)
- `docs/specs/ipc/signal/diagnostics.md` (3 responses)

These files follow the same pattern as the ones updated in this report and should be converted using the same approach.

## Impact

- **Consistency:** All updated specifications now follow a consistent pattern for command/response pairs
- **Clarity:** The distinction between direct responses (kind="response") and events (kind="event") is now explicit
- **Implementation:** Implementations can now rely on the rule that response `name` always matches command `name`

## Notes

- The `client.md` domain specification was already correctly following the response naming rules and required no changes
- All changes maintain backward compatibility in terms of message semantics; only the documentation and envelope format have been updated
- The envelope specification now explicitly documents the response naming rules for future reference

