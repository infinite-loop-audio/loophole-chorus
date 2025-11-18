# Architecture Folder Renumbering Report

This report documents the complete reorganisation and renumbering of the `docs/architecture/` folder completed on this date.

## Files Moved/Renamed

The following files were renamed to match the new architecture hierarchy:

1. **02-signal.md** → **03-signal.md**
   - No content changes, only filename update

2. **03-pulse.md** → **02-pulse.md**
   - No content changes, only filename update

3. **06-tracks-channels-and-lanes.md** → **11-tracks-lanes-and-roles.md**
   - Title updated: "Tracks, Channels and Lanes" → "Tracks, Lanes & Roles Architecture"
   - Content preserved exactly

4. **07-clips.md** → **12-clips.md**
   - No content changes, only filename update

5. **08-parameters.md** → **13-parameters.md**
   - No content changes, only filename update

6. **09-nodes.md** → **14-node-graph.md**
   - Title updated: "Nodes Architecture" → "Node Graph Architecture"
   - Content preserved exactly

7. **13-media-and-library.md** → **16-media-architecture.md**
   - Title updated: "Media, Library and Discovery Architecture" → "Media Architecture"
   - Content preserved exactly

8. **14-clip-launcher-and-scenes.md** → **23-clip-launcher.md**
   - Title updated: "Clip Launcher & Scenes Architecture" → "Clip Launcher Architecture"
   - Content preserved exactly

9. **12-plugin-ui-and-window-layout.md** → **27-windowing-and-plugin-ui.md**
   - Title updated: "Plugin UI and Window Layout Architecture" → "Windowing & Plugin UI Architecture"
   - Content preserved exactly

## Files Merged

10. **11-mixing-console.md** + routing architecture → **15-mixer-and-channel-architecture.md**
   - Title updated: "Mixing Model and Console Architecture" → "Mixer & Channel Architecture"
   - Routing architecture content added as new section 10 "Routing Architecture"
   - All original mixer content preserved
   - Routing content derived from conceptual routing architecture (no separate 10-routing.md existed in architecture folder)

## Files Deleted

The following obsolete files were removed after being replaced by new files:

1. **11-mixing-console.md** (merged into 15-mixer-and-channel-architecture.md)

Note: Files 06, 07, 08, 09, 10, 12, 13, 14 were renamed rather than deleted, so they appear in the "Files Moved/Renamed" section above.

## Merge Operation Summary

The mixer document (11-mixing-console.md) was merged with routing architecture concepts to create the new unified document (15-mixer-and-channel-architecture.md). The routing content was added as section 10 "Routing Architecture", covering:

- Overview of routing graph
- Routing concepts (Channels, Busses, Sends, Sidechains, Hardware I/O)
- Channel-to-Channel routing
- Sends and Returns
- Sidechain routing
- Hardware I/O mapping
- Routing graph and project structure

All original mixer content was preserved without modification. The routing section was written based on routing concepts from the Pulse Routing Domain Specification, adapted to architectural rather than IPC-focused content.

## Link Updates

All cross-links throughout the repository were updated to point to the new filenames. Updates were made in:

### Architecture Documents
- 01-overview.md (reference to processing cohorts - unchanged as file number didn't change)
- 05-composer.md (updated reference to parameters document)
- 11-tracks-lanes-and-roles.md (updated reference to node graph)
- 14-node-graph.md (updated reference to parameters)
- 15-mixer-and-channel-architecture.md (updated all internal references)
- 27-windowing-and-plugin-ui.md (updated reference to tracks document)

### Specification Documents
- docs/specs/ipc/pulse/channel.md (updated reference to mixer architecture)
- docs/specs/ipc/pulse/node.md (updated reference to mixer architecture)
- docs/specs/ipc/pulse/automation.md (updated references to parameters and mixer)
- docs/specs/ipc/pulse/parameter.md (updated references to parameters and mixer)
- docs/specs/ipc/pulse/routing.md (updated reference to mixer architecture)
- docs/specs/ipc/pulse/plugin-ui.md (updated reference to windowing document)

**Total link updates: 15+ cross-references updated across the repository**

## Naming Consistency

Checked for "processor" → "node" naming inconsistencies. All instances of "processor" found were in correct contexts:
- "signal processors" (referring to audio processing hardware/software)
- "dynamic processors" (referring to compressors, gates, etc.)

No architectural "processor" → "node" corrections were needed.

## Content Preservation

**No conceptual content was altered.** All changes were limited to:
- Filename updates
- Title heading updates (to match new file names)
- Cross-reference link updates
- Addition of routing section to mixer document (new content, not modification of existing)

All original architectural concepts, responsibilities, and relationships remain unchanged.

## Final Structure

The architecture folder now follows this structure:

```
docs/architecture/
01-overview.md
02-pulse.md
03-signal.md
04-aura.md
05-composer.md
10-processing-cohorts-and-anticipative-rendering.md
11-tracks-lanes-and-roles.md
12-clips.md
13-parameters.md
14-node-graph.md
15-mixer-and-channel-architecture.md
16-media-architecture.md
23-clip-launcher.md
27-windowing-and-plugin-ui.md
```

## Confirmation

✓ All files moved/renamed successfully
✓ All titles updated to match new filenames
✓ All cross-links updated throughout repository
✓ Merge operation completed (mixer + routing)
✓ Obsolete files deleted
✓ No conceptual content altered
✓ Naming consistency verified
✓ British English maintained
✓ No trailing newlines added

