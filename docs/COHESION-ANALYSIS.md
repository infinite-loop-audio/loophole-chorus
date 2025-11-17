# Cross-Document Cohesion & Consistency Analysis

**Date:** 2025-01-XX  
**Scope:** Full review of Loophole architecture and IPC documentation  
**Status:** In Progress

---

## Document Map

### Architecture Documents (`docs/architecture/`)

| File | Purpose | Domain | Dependencies |
|------|---------|--------|--------------|
| `01-overview.md` | System-wide architectural overview | All | None (foundational) |
| `02-signal.md` | Signal (audio engine) responsibilities | Signal | Overview, Processing Cohorts |
| `03-pulse.md` | Pulse (model layer) responsibilities | Pulse | Overview, Processing Cohorts |
| `04-aura.md` | Aura (UI layer) responsibilities | Aura | Overview |
| `05-composer.md` | Composer (metadata service) responsibilities | Composer | Overview |
| `06-tracks-channels-and-lanes.md` | Track/Channel/Lane structure | Pulse | Overview, Nodes |
| `07-clips.md` | Clip model and lifecycle | Pulse | Tracks/Channels/Lanes |
| `08-parameters.md` | Parameter identity and mapping | Pulse | Nodes, Composer |
| `09-nodes.md` | Node model and graph structure | Pulse/Signal | Channels, Parameters |
| `10-processing-cohorts-and-anticipative-rendering.md` | Dual-engine architecture | Pulse/Signal | All domains |
| `11-mixing-console.md` | Mixing model and console | Pulse/Signal | Channels, Nodes, Parameters |
| `12-plugin-ui-and-window-layout.md` | Plugin UI window management | Aura/Signal | Nodes |

### IPC Specifications (`docs/specs/ipc/`)

| File | Purpose | Domain | Dependencies |
|------|---------|--------|--------------|
| `overview.md` | IPC communication model | All | Envelope, Semantics |
| `envelope.md` | Message envelope format | All | None (foundational) |
| `semantics.md` | IPC semantic guidelines | All | Envelope |
| `pulse/track.md` | Track domain IPC | Pulse | Envelope, Architecture |
| `pulse/clip.md` | Clip domain IPC | Pulse | Envelope, Architecture |
| `pulse/lane.md` | Lane domain IPC | Pulse | Envelope, Architecture |
| `pulse/channel.md` | Channel domain IPC | Pulse | Envelope, Architecture, Nodes |
| `pulse/node.md` | Node domain IPC | Pulse | Envelope, Architecture, Channels |
| `pulse/parameter.md` | Parameter domain IPC | Pulse | Envelope, Architecture |
| `pulse/automation.md` | Automation domain IPC | Pulse | Envelope, Architecture, Parameters |
| `pulse/project.md` | Project lifecycle IPC | Pulse | Envelope, Architecture |
| `pulse/transport.md` | Transport control IPC | Pulse | Envelope, Architecture |
| `pulse/media.md` | Media pool IPC | Pulse | Envelope, Architecture |
| `pulse/rendering.md` | Rendering/bounce IPC | Pulse | Envelope, Architecture |
| `pulse/recording.md` | Recording IPC | Pulse | Envelope, Architecture, Media |
| `pulse/routing.md` | Routing IPC | Pulse | Envelope, Architecture, Channels |
| `pulse/timebase.md` | Timebase/tempo IPC | Pulse | Envelope, Architecture |
| `pulse/history.md` | Undo/redo IPC | Pulse | Envelope, Architecture |
| `pulse/gesture.md` | Gesture handling IPC | Pulse | Envelope, Architecture, Parameters |
| `pulse/hardware-io.md` | Hardware I/O IPC | Pulse/Signal | Envelope, Architecture |
| `pulse/plugin-ui.md` | Plugin UI lifecycle IPC | Pulse/Signal/Aura | Envelope, Architecture |
| `pulse/metering.md` | Metering IPC | Pulse/Signal | Envelope, Architecture |
| `pulse/engine-diagnostics.md` | Engine diagnostics IPC | Signal | Envelope, Architecture |
| `pulse/session.md` | Session management IPC | Pulse | Envelope, Architecture |
| `pulse/project-metadata.md` | Project metadata IPC | Pulse | Envelope, Architecture |

### Guidelines (`docs/specs/guidelines/`)

| File | Purpose | Domain | Dependencies |
|------|---------|--------|--------------|
| `realtime-safety.md` | RT safety constraints | Signal | Architecture |
| `versioning.md` | Versioning strategy | All | None |

### Meta Documents (`docs/meta/`)

| File | Purpose | Domain | Dependencies |
|------|---------|--------|--------------|
| `glossary.md` | Terminology definitions | All | None |

### ADRs (`docs/decisions/`)

| File | Purpose | Domain | Dependencies |
|------|---------|--------|--------------|
| `0001-initial-architecture.md` | Multi-process architecture decision | All | None |
| `0002-ipc-transport-and-topology.md` | IPC transport decision | All | ADR 0001 |

---

## Issues Identified

### 1. ID Schema Consistency

#### Issue 1.1: Mixed ID Naming Patterns
**Location:** Multiple files  
**Severity:** Medium  
**Status:** ✅ FIXED

**Problem:**
- Most IDs use camelCase: `trackId`, `laneId`, `nodeId`, `channelId`, `mediaId`
- Some use different patterns: `clipId` (in architecture) vs `clipId` (in IPC)
- Parameter IDs use dot notation: `track.12.channel.gain`
- Plugin UI window keys use different format: `plugin-[trackId]-[nodeId]`

**Examples:**
- `docs/architecture/07-clips.md`: Uses `id` and `hostTrackId`
- `docs/specs/ipc/pulse/clip.md`: Uses `clipId` and `trackId`
- `docs/architecture/08-parameters.md`: Uses `entityType.entityId.nodeId.parameterId`
- `docs/architecture/12-plugin-ui-and-window-layout.md`: Uses `plugin-[trackId]-[nodeId]`

**Fix Applied:**
- Updated `docs/architecture/07-clips.md` to use `clipId` consistently instead of `id`
- Clarified that `hostTrackId` is an alias for `trackId` in some contexts
- Parameter fully-qualified IDs remain as dot-notation (intentional for hierarchical addressing)

---

### 2. Domain Responsibility Boundaries

#### Issue 2.1: Channel Ownership Ambiguity
**Location:** `docs/architecture/06-tracks-channels-and-lanes.md` vs `docs/architecture/03-pulse.md`  
**Severity:** High  
**Status:** ✅ FIXED

**Problem:**
- `06-tracks-channels-and-lanes.md` line 44: "Channels: live exclusively in Signal"
- `03-pulse.md` line 89: "Pulse constructs the node graph" and owns Channel structure
- `docs/specs/ipc/pulse/channel.md` treats Channels as Pulse-owned

**Reality:**
- Pulse owns Channel *structure* and *configuration*
- Signal owns Channel *execution* and *runtime state*
- This is actually correct but the wording in `06-tracks-channels-and-lanes.md` is misleading

**Fix Applied:**
- Updated `06-tracks-channels-and-lanes.md` to explicitly distinguish "Channel Model (Pulse)" from "Channel Runtime (Signal)"
- Clarified that Pulse configures Channels and sends instructions to Signal, which executes them

---

#### Issue 2.2: LaneStream Node Ownership
**Location:** Multiple files  
**Severity:** Medium  
**Status:** ✅ FIXED

**Problem:**
- `docs/architecture/09-nodes.md` describes LaneStreams as nodes
- `docs/architecture/06-tracks-channels-and-lanes.md` describes LaneStreams as Channel processors
- `docs/specs/ipc/pulse/node.md` includes LaneStreamNode as a node type
- But `docs/specs/ipc/pulse/lane.md` mentions `laneStreamId` without clear definition

**Fix Applied:**
- Updated `06-tracks-channels-and-lanes.md` to clarify LaneStreamNode as a first-class Node type
- Updated `docs/specs/ipc/pulse/lane.md` to clarify that `laneStreamId` refers to the `nodeId` of a LaneStreamNode
- Added cross-reference to Node domain specification

---

### 3. Lifecycle Consistency

#### Issue 3.1: Media Reserve/Finalise Semantics
**Location:** `docs/specs/ipc/pulse/media.md`  
**Severity:** Low

**Problem:**
- `media.reserveRecordingTarget` creates a Media Item with status `pending`
- `media.finaliseRecording` marks it complete
- But the lifecycle of "pending" media items during project save/load is unclear

**Proposed Fix:**
- Document that pending media items are not included in project snapshots
- Clarify cleanup behaviour for abandoned recordings

---

#### Issue 3.2: Snapshot Application Semantics
**Location:** Multiple IPC domain specs  
**Severity:** Medium

**Problem:**
- Each domain spec mentions "snapshot semantics" but the *application* of snapshots is not consistently described
- Some domains say "Aura discards conflicting local state" but don't specify when snapshots are applied vs incremental updates

**Proposed Fix:**
- Add consistent section to each domain spec describing snapshot application behaviour
- Clarify that snapshots are authoritative replacements, not merges

---

### 4. Referential Consistency

#### Issue 4.1: Broken Cross-References
**Location:** Multiple files  
**Severity:** Low  
**Status:** ✅ FIXED

**Problem:**
- `docs/specs/ipc/pulse/parameter.md` line 67: References `11-mixing-model-and-console-architecture.md` (wrong filename)
- `docs/specs/ipc/pulse/automation.md` line 67: References `11-mixing-model-and-console-architecture.md` (wrong filename)

**Actual filename:** `11-mixing-console.md`

**Fix Applied:**
- Corrected all references in `parameter.md` and `automation.md` to use `11-mixing-console.md`

---

#### Issue 4.2: Inconsistent File Reference Format
**Location:** Multiple files  
**Severity:** Low

**Problem:**
- Some files use `@chorus:/docs/...` format
- Others use relative paths like `../../architecture/...`
- Some use just filenames like `08-parameters.md`

**Proposed Fix:**
- Standardise on relative paths within the same repo
- Use `@chorus:/docs/...` only for cross-repo references (if any)

---

### 5. Envelope Compatibility

#### Issue 5.1: Domain Enumeration
**Location:** `docs/specs/ipc/envelope.md`  
**Severity:** Low  
**Status:** ✅ FIXED

**Problem:**
- Envelope spec lists domains but some IPC domain specs exist that aren't listed
- Missing: `gesture`, `metering`, `engine`, `cohort` (mentioned in envelope but not all domains are listed)

**Fix Applied:**
- Updated envelope domain enumeration to include all actual domain specs:
  - Added: `timebase`, `media`, `recording`, `rendering`, `gesture`, `metering`, `pluginUi`, `session`, `history`, `hardwareIo`, `engineDiagnostics`, `projectMetadata`

---

### 6. Terminology Consistency

#### Issue 6.1: "Processing Cohort" vs "Cohort"
**Location:** Multiple files  
**Severity:** Low

**Problem:**
- Some files use "Processing Cohort" (full term)
- Others use "cohort" (short form)
- Some use "Processing Cohorts" (plural)

**Proposed Fix:**
- Standardise: Use "processing cohort" (lowercase) for the concept
- Use "Processing Cohorts" (title case) only when referring to the architecture document title
- Use "cohort" as acceptable shorthand after first use

---

#### Issue 6.2: "LaneStream" vs "LaneStream Node"
**Location:** Multiple files  
**Severity:** Low

**Problem:**
- Some files say "LaneStream node"
- Others say "LaneStreamNode"
- Some say "LaneStream" (implying it's not a node)

**Proposed Fix:**
- Standardise: "LaneStreamNode" (one word) when referring to the node type
- "LaneStream" (two words) is acceptable as shorthand after first use
- Ensure all references clarify that LaneStreams are nodes

---

### 7. Missing Specifications

#### Issue 7.1: Clip ID Field Name
**Location:** `docs/architecture/07-clips.md`  
**Severity:** Medium

**Problem:**
- Architecture doc uses `id` for Clip identifier
- IPC spec uses `clipId`
- No explicit mapping documented

**Proposed Fix:**
- Update architecture doc to use `clipId` consistently
- Or document that `id` in architecture maps to `clipId` in IPC

---

#### Issue 7.2: Automation Lane Creation
**Location:** `docs/specs/ipc/pulse/automation.md`  
**Severity:** Low

**Problem:**
- Automation spec says "creation of the underlying automation lane... is handled by the Lane domain"
- But Lane domain doesn't explicitly document automation lane creation workflow

**Proposed Fix:**
- Add explicit automation lane creation example to Lane domain spec
- Or add cross-reference with workflow description

---

## Summary of Fixes Required

### High Priority
1. Clarify Channel ownership (Pulse owns structure, Signal owns execution)
2. Fix broken cross-references to mixing console document

### Medium Priority
1. Standardise ID field naming (camelCase throughout)
2. Clarify LaneStream as Node with consistent terminology
3. Document Clip ID field mapping (`id` vs `clipId`)
4. Add consistent snapshot application semantics to all domain specs

### Low Priority
1. Standardise file reference formats
2. Update envelope domain enumeration
3. Standardise terminology (cohort, LaneStream)
4. Document media lifecycle for pending items
5. Add automation lane creation workflow

---

## Next Steps

1. Review and approve proposed fixes
2. Apply fixes systematically, one file per commit
3. Generate final cohesion report with dependency graph
4. Create consolidated glossary updates

---

*This analysis is ongoing. Additional issues may be discovered during fix application.*

