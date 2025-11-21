# IPC Event Name Normalisation Report

**Date:** 2025-11-21  
**Time:** 23:01:50  
**Status:** Completed  
**Scope:** Chorus IPC specification documentation — event name domain prefix removal

## Summary

This report documents the normalisation of IPC event names across all Pulse domain specifications in the Chorus repository. The goal was to ensure that event names follow a consistent pattern where:

- **No domain prefix duplication** in the `name` field
- **Domain is authoritative**, `name` is domain-relative
- **CamelCase** names are preserved where appropriate
- **Only `kind: "event"`** names were changed, not commands or responses

## Event Name Mapping

### Track Domain
- `track.created` → `created`
- `track.deleted` → `deleted`
- `track.moved` → `moved`
- `track.renamed` → `renamed`
- `track.colourChanged` → `colourChanged`
- `track.iconChanged` → `iconChanged`
- `track.flagsChanged` → `flagsChanged`
- `track.processingPolicyChanged` → `processingPolicyChanged`
- `track.channelBindingChanged` → `channelBindingChanged`

**Note:** Added `updated` event to represent general track updates (e.g. rename, parent change).

### Project Domain
- `project.loaded` → `loaded`
- `project.loadFailed` → `loadFailed`
- `project.closed` → `closed`
- `project.draftSaved` → `draftSaved`
- `project.draftSaveFailed` → `draftSaveFailed`
- `project.saved` → `saved`
- `project.saveFailed` → `saveFailed`
- `project.renamed` → `renamed`
- `project.renameFailed` → `renameFailed`
- `project.moved` → `moved`
- `project.moveFailed` → `moveFailed`
- `project.metaUpdated` → `metaUpdated`
- `project.metaUpdateFailed` → `metaUpdateFailed`

**Note:** `project.snapshot` remains as `snapshot` with `kind: "snapshot"` (not an event).

### Channel Domain
- `channel.created` → `created`
- `channel.deleted` → `deleted`
- `channel.inputChanged` → `inputChanged`
- `channel.outputChanged` → `outputChanged`
- `channel.sidechainSourceChanged` → `sidechainSourceChanged`
- `channel.gainChanged` → `gainChanged`
- `channel.panChanged` → `panChanged`
- `channel.meteringConfigChanged` → `meteringConfigChanged`
- `channel.graphReset` → `graphReset`

### Node Domain
- `node.added` → `added`
- `node.removed` → `removed`
- `node.moved` → `moved`
- `node.enabledChanged` → `enabledChanged`
- `node.bypassChanged` → `bypassChanged`
- `node.capabilitiesChanged` → `capabilitiesChanged`
- `node.faulted` → `faulted`
- `node.metadataChanged` → `metadataChanged`
- `node.activeVariantChanged` → `activeVariantChanged`
- `node.variantAdded` → `variantAdded`
- `node.variantRemoved` → `variantRemoved`
- `node.variantMetadataChanged` → `variantMetadataChanged`

### Automation Domain
- `automation.envelopeCreated` → `envelopeCreated`
- `automation.envelopeDeleted` → `envelopeDeleted`
- `automation.pointAdded` → `pointAdded`
- `automation.pointUpdated` → `pointUpdated`
- `automation.pointDeleted` → `pointDeleted`
- `automation.rangeChanged` → `rangeChanged`
- `automation.modeChanged` → `modeChanged`
- `automation.writeArmedChanged` → `writeArmedChanged`

### Clip Domain
- `clip.created` → `created`
- `clip.deleted` → `deleted`
- `clip.moved` → `moved`
- `clip.resized` → `resized`
- `clip.loopingChanged` → `loopingChanged`
- `clip.stretchChanged` → `stretchChanged`
- `clip.slipped` → `slipped`
- `clip.laneAdded` → `laneAdded`
- `clip.laneRemoved` → `laneRemoved`
- `clip.lanesReordered` → `lanesReordered`

### Lane Domain
- `lane.created` → `created`
- `lane.deleted` → `deleted`
- `lane.attachedToClip` → `attachedToClip`
- `lane.detachedFromClip` → `detachedFromClip`
- `lane.reorderedInClip` → `reorderedInClip`
- `lane.renamed` → `renamed`
- `lane.colourChanged` → `colourChanged`
- `lane.collapsedChanged` → `collapsedChanged`
- `lane.laneStreamChanged` → `laneStreamChanged`
- `lane.sendLevelChanged` → `sendLevelChanged`

### Routing Domain
- `routing.busCreated` → `busCreated`
- `routing.busDeleted` → `busDeleted`
- `routing.masterChannelChanged` → `masterChannelChanged`
- `routing.sendAdded` → `sendAdded`
- `routing.sendRemoved` → `sendRemoved`
- `routing.sendUpdated` → `sendUpdated`
- `routing.sidechainRouteAdded` → `sidechainRouteAdded`
- `routing.sidechainRouteRemoved` → `sidechainRouteRemoved`
- `routing.hardwareOutputMappingChanged` → `hardwareOutputMappingChanged`
- `routing.hardwareInputMappingChanged` → `hardwareInputMappingChanged`

### Domains That Were Already Correct

The following domains already used domain-relative event names and required no changes:

- **Client domain:** `welcome`, `connected`, `disconnected`, `reconnected`, `timedOut`, `subscriptionsUpdated`, `capabilitiesUpdated`
- **Transport domain:** `stateChanged`, `error`
- **Parameter domain:** `valueChanged`, `published`, `gestureBegan`, `gestureEnded`, `groupCreated`, `groupDeleted`, `groupMembersChanged`
- **Gesture domain:** `streamCreated`, `streamDestroyed`, `parameterBound`, `parameterUnbound`, `dataChannelReady`, `dataChannelFailed`
- **Debug domain:** `log`, `diagnostic`, `stateChunk`

## Files Modified

### Domain Specifications
1. `docs/specs/ipc/pulse/track.md`
2. `docs/specs/ipc/pulse/project.md`
3. `docs/specs/ipc/pulse/channel.md`
4. `docs/specs/ipc/pulse/node.md`
5. `docs/specs/ipc/pulse/automation.md`
6. `docs/specs/ipc/pulse/clip.md`
7. `docs/specs/ipc/pulse/lane.md`
8. `docs/specs/ipc/pulse/routing.md`

All event name references were updated to use domain-relative names with explicit "(event — domain: X)" annotations for clarity.

## Notes

- **Commands and responses** were not modified; they already follow the correct naming pattern (domain-relative, no prefixes).
- **Snapshots** (`kind: "snapshot"`) were not changed; they follow their own naming conventions.
- Event names that did not have domain prefixes (e.g. `stateChanged` in transport domain) were left unchanged.
- All updated event names maintain camelCase formatting.
- References to `project.snapshot` were updated to clarify it is `kind: "snapshot"` not `kind: "event"`.

## Verification

- All event names now follow the pattern: domain-relative, camelCase, no redundant prefixes.
- No commands or responses were modified.
- No files in `docs/reports/` were modified (only this new report was added).
- All domain specifications consistently use the new naming pattern.

