# IPC Spec File Reorganization Report

**Date:** 2025-01-XX  
**Scope:** Loophole Chorus IPC specification files

## Summary

This report documents the reorganization of IPC specification files in the Loophole Chorus repository. The goal was to make filenames shorter, clearer, and less redundant by using directory structure to encode ownership (Pulse/Signal/Aura/Composer) rather than repeating it in filenames.

## Changes Made

### 1. Directory Structure Changes

**Created new directories:**
- `docs/specs/ipc/pulse/` - For Pulse-specific domain specifications

### 2. File Moves and Renames

#### Core IPC Files

| Old Path | New Path | Reason |
|----------|----------|--------|
| `docs/specs/ipc-envelope.md` | `docs/specs/ipc/envelope.md` | Removed redundant "ipc-" prefix; moved into ipc/ directory |
| `docs/specs/guidelines/ipc-overview.md` | `docs/specs/ipc/overview.md` | Moved to ipc/ root; removed "ipc-" prefix |
| `docs/specs/guidelines/ipc-semantics.md` | `docs/specs/ipc/semantics.md` | Moved to ipc/ root; removed "ipc-" prefix |

#### Pulse Domain Files

All Pulse domain files were moved from `docs/specs/ipc/` to `docs/specs/ipc/pulse/` and renamed to remove the "pulse-" prefix and "-domain" suffix:

| Old Path | New Path | Reason |
|----------|----------|--------|
| `docs/specs/ipc/pulse-project-domain.md` | `docs/specs/ipc/pulse/project.md` | Ownership encoded in directory; removed redundant prefixes |
| `docs/specs/ipc/pulse-track-domain.md` | `docs/specs/ipc/pulse/track.md` | Ownership encoded in directory; removed redundant prefixes |
| `docs/specs/ipc/pulse-clip-domain.md` | `docs/specs/ipc/pulse/clip.md` | Ownership encoded in directory; removed redundant prefixes |
| `docs/specs/ipc/pulse-lane-domain.md` | `docs/specs/ipc/pulse/lane.md` | Ownership encoded in directory; removed redundant prefixes |
| `docs/specs/ipc/pulse-transport-domain.md` | `docs/specs/ipc/pulse/transport.md` | Ownership encoded in directory; removed redundant prefixes |

### 3. Reference Updates

**Files updated with new paths:**
- `docs/meta/meta-commands.md` - Updated example reference from `docs/specs/ipc-semantics.md` to `docs/specs/ipc/semantics.md`
- `docs/specs/README.md` - Updated directory structure example to reflect new organization

**Files with directory references (no changes needed):**
- `docs/specs/ipc/overview.md` - References `@chorus:/docs/specs/ipc/` (still valid)
- `docs/specs/ipc/semantics.md` - References `@chorus:/docs/specs/ipc/` (still valid)
- `docs/decisions/0002-ipc-transport-and-topology.md` - References `@chorus:/docs/specs/ipc/` (still valid)
- `aura/README.md` - References `@chorus:/docs/specs/ipc/` (still valid)

## Final File Structure

```
docs/specs/ipc/
├── envelope.md
├── overview.md
├── semantics.md
└── pulse/
    ├── clip.md
    ├── lane.md
    ├── project.md
    ├── track.md
    └── transport.md
```

## Benefits

1. **Shorter, clearer filenames** - Removed redundant prefixes ("pulse-", "ipc-") and suffixes ("-domain")
2. **Better organization** - Ownership (Pulse) is encoded in directory structure rather than filenames
3. **Scalability** - Easy to add Signal, Aura, or Composer-specific specs in their own directories
4. **Consistency** - All IPC-related files are now under `docs/specs/ipc/`
5. **Maintainability** - Clearer structure makes it easier to find and maintain files

## Migration Notes

- All internal links in the Chorus repository have been updated
- Cross-repo references to `@chorus:/docs/specs/ipc/` remain valid (directory-level references)
- No breaking changes to content - only file locations and names changed
- Old files have been removed after successful migration

## Future Structure

The new structure supports future additions:

```
docs/specs/ipc/
├── envelope.md
├── overview.md
├── semantics.md
├── pulse/
│   ├── clip.md
│   ├── lane.md
│   ├── project.md
│   ├── track.md
│   └── transport.md
├── signal/          # Future: Signal-specific specs
│   └── ...
├── aura/            # Future: Aura-specific specs
│   └── ...
└── composer/        # Future: Composer-specific specs
    └── ...
```

## Verification

All files have been:
- ✅ Moved to new locations
- ✅ Renamed appropriately
- ✅ References updated
- ✅ Old files removed
- ✅ Directory structure verified

---

**Status:** Complete  
**Files Changed:** 8 files moved/renamed, 2 reference files updated, 8 old files deleted

