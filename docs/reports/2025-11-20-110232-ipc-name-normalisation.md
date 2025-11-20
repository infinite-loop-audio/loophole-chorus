# IPC Name Normalisation Migration Report

**Date:** 2025-11-20  
**Status:** In Progress  
**Scope:** Chorus IPC specification documentation — dotted names → camelCase

## Summary

This report documents the migration of IPC message names from dotted notation (e.g. `parameter.gesture.begin`) to camelCase (e.g. `gestureBegin`) across the Chorus IPC specification documentation. The `name` field in envelopes is now always unqualified camelCase, with the `domain` field providing the namespace.

## Completed Files

### Core IPC Documentation

1. **`docs/specs/ipc/envelope.md`**
   - Updated prose references to dotted names
   - Clarified that fully-qualified names are derived for logging only
   - Updated example references

2. **`docs/specs/ipc/overview.md`**
   - Converted `gesture.begin` → `beginGesture`
   - Converted `gesture.end` → `endGesture`
   - Updated prose references

### Pulse Domain Specifications

3. **`docs/specs/ipc/pulse/client.md`**
   - Converted all section headers: `#### `client.hello`` → `#### hello (command — domain: client)`
   - Updated all prose references from `client.hello` to `hello` (with domain context)
   - Converted: `hello`, `unregister`, `heartbeat`, `subscribeEvents`, `unsubscribeEvents`, `updateCapabilities`, `updatePreferences`
   - Converted events: `welcome`, `registered`, `unregistered`, `reconnected`, `heartbeatRequired`, `timedOut`, `subscriptionsUpdated`, `capabilitiesUpdated`

4. **`docs/specs/ipc/pulse/debug.md`**
   - Converted all section headers to new format
   - Converted: `setLogLevel`, `enableDomainLogging`, `getStateSummary`, `getDomainState`, `echo`, `simulateError`
   - Converted events: `log`, `diagnostic`, `stateChunk`
   - Updated error message examples

5. **`docs/specs/ipc/pulse/parameter.md`**
   - Converted bold message names to section headers
   - Converted commands: `setValue`, `beginGesture`, `updateGesture`, `endGesture`, `createGroup`, `deleteGroup`, `setGroupMembers`, `setGroupMasterValue`, `setPresentation`
   - Converted events: `valueChanged`, `published`, `gestureBegan`, `gestureEnded`, `groupCreated`, `groupDeleted`, `groupMembersChanged`
   - Updated prose references and snapshot semantics

6. **`docs/specs/ipc/pulse/gesture.md`**
   - Converted all message names to camelCase
   - Converted commands: `createStream`, `destroyStream`, `bindParameter`, `unbindParameter`, `negotiateDataChannel`, `beginGesture`, `endGesture`
   - Converted events: `streamCreated`, `streamDestroyed`, `parameterBound`, `parameterUnbound`, `dataChannelReady`, `dataChannelFailed`

7. **`docs/specs/ipc/pulse/transport.md`**
   - Converted all message names
   - Converted commands: `play`, `stop`, `seek`, `setLoopRegion`, `setLoopEnabled`
   - Converted events: `stateChanged`, `error`
   - Updated prose references

8. **`docs/specs/ipc/pulse/project.md`** (partial)
   - Converted commands: `new`, `open`, `close`, `saveDraft`, `save`, `saveAs`, `rename`, `move`, `updateMeta`
   - Updated prose references to `snapshot` events
   - **Note:** Events section and remaining commands still need conversion

## Conversion Patterns Applied

### Section Headers

**Before:**
```markdown
#### `client.hello`
```

**After:**
```markdown
#### hello (command — domain: client)
```

or for events:
```markdown
#### welcome (event — domain: client)
```

### Bold Message Names

**Before:**
```markdown
**`parameter.setValue`**  
Set the value of a parameter directly.
```

**After:**
```markdown
#### setValue (command — domain: parameter)

Set the value of a parameter directly.
```

### Prose References

**Before:**
```markdown
Pulse may also emit `client.welcome` and `client.registered` events.
```

**After:**
```markdown
Pulse may also emit `welcome` and `registered` events.
```

### Multi-word Names

**Before:**
```markdown
`parameter.gesture.begin` → `gestureBegin`
`transport.state.changed` → `transportStateChanged`
`parameter.changed.value` → `parameterChangedValue`
```

**After:**
```markdown
`beginGesture` (domain: parameter)
`stateChanged` (domain: transport)
`parameterChangedValue` (domain: parameter)
```

### Snapshot References

**Before:**
```markdown
When Aura receives a `project.snapshot` that includes this domain...
```

**After:**
```markdown
When Aura receives a `snapshot` event (domain: project) that includes this domain...
```

## Remaining Work

### Pulse Domain Files (17 files remaining)

The following files still contain dotted message names and need conversion:

1. `docs/specs/ipc/pulse/channel.md` (15 matches)
2. `docs/specs/ipc/pulse/node.md` (24 matches)
3. `docs/specs/ipc/pulse/track.md` (19 matches)
4. `docs/specs/ipc/pulse/automation.md` (20 matches)
5. `docs/specs/ipc/pulse/routing.md` (23 matches)
6. `docs/specs/ipc/pulse/hardware-io.md` (17 matches)
7. `docs/specs/ipc/pulse/clip.md` (25 matches)
8. `docs/specs/ipc/pulse/lane.md` (20 matches)
9. `docs/specs/ipc/pulse/recording.md` (18 matches)
10. `docs/specs/ipc/pulse/rendering.md` (13 matches)
11. `docs/specs/ipc/pulse/timebase.md` (18 matches)
12. `docs/specs/ipc/pulse/session.md` (10 matches)
13. `docs/specs/ipc/pulse/project.md` (events section remaining)
14. `docs/specs/ipc/pulse/project-metadata.md` (37 matches)
15. `docs/specs/ipc/pulse/media.md` (22 matches)
16. `docs/specs/ipc/pulse/history.md` (11 matches)
17. `docs/specs/ipc/pulse/metering.md` (12 matches)

**Total:** ~337 message name references remaining across these files.

### Signal Domain Files

All Signal domain specification files need conversion:
- `docs/specs/ipc/signal/*.md`

### Other IPC Documentation

- `docs/specs/ipc/ipc-semantics.md` (if it exists and contains message names)
- `docs/specs/ipc/versioning.md` (if it exists and contains message names)
- Any architecture files under `docs/architecture/` that reference IPC message names

## Conversion Guidelines for Remaining Files

### Step 1: Identify Message Names

Search for patterns:
- `**`domain.messageName`` (bold formatting)
- `#### `domain.messageName`` (section headers)
- `` `domain.messageName` `` (inline code references)

### Step 2: Convert to camelCase

Rules:
- Remove all dots
- Capitalise first letter after each removed dot
- Keep domain prefix only if it's part of the semantic name (e.g. `parameterGestureBegin` if the domain is not "parameter")
- Otherwise, remove domain prefix entirely (e.g. `gestureBegin` when domain is "parameter")

Examples:
- `parameter.gesture.begin` → `beginGesture` (domain: parameter)
- `transport.state.changed` → `stateChanged` (domain: transport)
- `node.add` → `add` (domain: node)
- `track.channel.binding.changed` → `channelBindingChanged` (domain: track)

### Step 3: Update Section Headers

Convert:
```markdown
**`domain.messageName`**
```

to:
```markdown
#### messageName (command — domain: domain)
```

or:
```markdown
#### messageName (event — domain: domain)
```

### Step 4: Update Prose References

Replace all inline references:
- `` `domain.messageName` `` → `` `messageName` `` (with domain context in surrounding prose if needed)
- `domain.messageName` → `messageName` (in narrative text)

### Step 5: Update JSON Examples

Ensure all envelope examples use:
```json
{
  "domain": "parameter",
  "name": "beginGesture"
}
```

Not:
```json
{
  "domain": "parameter",
  "name": "parameter.gesture.begin"
}
```

### Step 6: Update Cross-References

Update any cross-document references to use the new camelCase names.

## Ambiguous Cases and Resolutions

### Case 1: Multi-level Dotted Names

**Pattern:** `parameter.gesture.begin`

**Resolution:** Convert to `beginGesture` (domain: parameter). The domain provides context, so we don't need `parameter` in the name.

**Rationale:** The domain field already identifies the namespace, so repeating it in the name is redundant.

### Case 2: Error Codes

**Pattern:** Error codes in `error.code` field (e.g. `"project.notFound"`)

**Resolution:** **Keep as-is.** Error codes are not envelope `name` fields. They may retain dotted notation for hierarchical organisation.

**Rationale:** Error codes are machine-readable identifiers, not envelope message names. They can use different conventions.

### Case 3: Parameter IDs

**Pattern:** Parameter identifiers like `"track.5.channel.proc.3.cutoff"`

**Resolution:** **Keep as-is.** These are identifiers, not envelope message names.

**Rationale:** Identifiers use hierarchical paths for uniqueness, which is different from message names.

## Verification Checklist

For each converted file, verify:

- [ ] All section headers use new format: `#### name (command/event — domain: domain)`
- [ ] All prose references use unqualified camelCase names
- [ ] All JSON envelope examples use `"name": "camelCase"` (not dotted)
- [ ] All cross-references updated
- [ ] No remaining dotted message names in code blocks or examples
- [ ] Error codes and identifiers (not message names) remain unchanged

## Notes

- This migration affects **documentation only**. Code implementations (Pulse, Aura, Signal) have already been updated separately.
- The fully-qualified form (`domain.name`) is still used in narrative examples and logging descriptions, but is never stored in envelope structures.
- All envelope structures now use separate `domain` and `name` fields, with `name` always being unqualified camelCase.

