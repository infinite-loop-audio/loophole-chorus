# Processor → Node Terminology Replacement Report

This report documents the comprehensive replacement of "Processor" terminology with "Node" across the Loophole documentation, reflecting the new terminology where **Nodes** replace **Processors** as the DSP graph element in Signal and Pulse.

---

## A. File Renames

### Renamed Files

1. `docs/architecture/09-processors.md` → `docs/architecture/09-nodes.md`

---

## B. Files Modified

### Architecture Documents
- `docs/architecture/01-overview.md`
- `docs/architecture/02-signal.md`
- `docs/architecture/03-pulse.md`
- `docs/architecture/04-aura.md`
- `docs/architecture/05-composer.md`
- `docs/architecture/06-tracks-channels-and-lanes.md`
- `docs/architecture/07-clips.md`
- `docs/architecture/08-parameters.md`
- `docs/architecture/10-processing-cohorts-and-anticipative-rendering.md`

### IPC Specification Files
- `docs/specs/ipc/envelope.md`
- `docs/specs/ipc/overview.md`
- `docs/specs/ipc/pulse/node.md`
- `docs/specs/ipc/pulse/lane.md`
- `docs/specs/ipc/pulse/track.md`
- `docs/specs/ipc/pulse/clip.md`

### Diagrams
- `docs/diagrams/processing-cohots.mermaid`

### Meta Files
- `docs/meta/architecture-inbox.md`

**Total Files Modified:** 17 files

---

## C. Summary of Textual Replacements

### Primary Replacements

1. **"Processor" → "Node"** (singular)
   - Applied throughout all DSP graph context references
   - Examples: "processor chain" → "node graph", "processor ID" → "node ID"

2. **"Processors" → "Nodes"** (plural)
   - Applied to all plural references to DSP graph elements
   - Examples: "processors in the chain" → "nodes in the graph"

3. **"Processor Chain" → "Node Graph"**
   - Updated all references to the serial processing structure
   - Examples: "processor chain construction" → "node graph construction"

4. **"Processor ID" → "Node ID"**
   - Updated all identifier references
   - Examples: "processorId" → "nodeId" in code examples and schemas

5. **"Processor Types" → "Node Types"**
   - Updated section headings and content
   - Examples: "Processor Types" → "Node Types"

6. **"Processor Lifecycle" → "Node Lifecycle"**
   - Updated lifecycle documentation sections

7. **"Processor Chain" → "Node Graph"** (in headings)
   - Updated section headings and cross-references

8. **"Deterministic vs Non-Deterministic Processors" → "Deterministic vs Non-Deterministic Nodes"**
   - Updated cohort assignment documentation

9. **"Plugin Processors" → "Plugin Nodes"**
   - Updated references to plugin-based nodes
   - Note: "plugin-based processors" in descriptive text (describing what plugins are) was left unchanged where appropriate

10. **Domain References**
    - `"processor"` domain → `"node"` domain in IPC envelope specifications
    - `processor.add` → `node.add` in message type examples
    - `processorId` → `nodeId` in JSON payload examples

11. **Parameter ID Format**
    - `entityType.entityId.processorId.parameterId` → `entityType.entityId.nodeId.parameterId`
    - Example: `track.5.processor.7.cutoff` → `track.5.node.7.cutoff`

12. **Specific Node Type References**
    - "LaneStream Processors" → "LaneStream Nodes"
    - "Instrument Processors" → "Instrument Nodes"
    - "Analyzer Processors" → "Analyzer Nodes"

### Terminology Preserved (Intentionally Not Changed)

The following uses of "processor" were **intentionally preserved** as they refer to:
- **CPU processors** (hardware)
- **Signal processors** (describing Channels as signal processing units, not the graph element)
- **Plugin-based processors** (describing the nature of plugins themselves in descriptive contexts)

---

## D. Final File Layout

### Architecture Directory
```
docs/architecture/
├── 01-overview.md
├── 02-signal.md
├── 03-pulse.md
├── 04-aura.md
├── 05-composer.md
├── 06-tracks-channels-and-lanes.md
├── 07-clips.md
├── 08-parameters.md
├── 09-nodes.md                    ← RENAMED from 09-processors.md
└── 10-processing-cohorts-and-anticipative-rendering.md
```

### IPC Specifications Directory
```
docs/specs/ipc/
├── envelope.md
├── overview.md
├── semantics.md
├── REORGANIZATION_REPORT.md
└── pulse/
    ├── clip.md
    ├── lane.md
    ├── node.md                    ← Already named correctly
    ├── project.md
    ├── track.md
    └── transport.md
```

### Diagrams Directory
```
docs/diagrams/
├── high-level-system-architecture.mermaid
├── processing-cohots.mermaid      ← Updated
└── track-channel-architecture.mermaid
```

### Meta Directory
```
docs/meta/
├── architecture-inbox.md          ← Updated
├── glossary.md
├── meta-commands.md
├── meta-protocol.md
└── meta-schema.json
```

---

## E. Validation Summary

### Remaining "Processor" References

After comprehensive replacement, the following uses of "processor" remain and are **intentionally preserved**:

1. **`docs/architecture/06-tracks-channels-and-lanes.md`**
   - "Channels are engine-level signal processors" - refers to Channels as signal processing units (acceptable)
   - "A LaneStream is a Channel processor node" - describes LaneStream as a processing node (acceptable)

2. **`docs/specs/ipc/pulse/node.md`**
   - "plugin-based processors (VST, CLAP, AU wrappers)" - describes what plugins are (acceptable)

All remaining uses refer to:
- General signal processing concepts (not the specific DSP graph element)
- Plugin characteristics (describing plugins themselves)
- Hardware/CPU processors (not DSP graph elements)

**No DSP graph element references to "Processor" remain.**

---

## F. Cross-Reference Updates

All internal cross-references have been updated:
- Section heading links updated
- File references updated
- Anchor fragments updated where applicable
- IPC domain references updated
- Parameter ID format examples updated

---

## G. Impact Assessment

### Breaking Changes
- IPC domain name changed: `"processor"` → `"node"`
- IPC message types changed: `processor.add` → `node.add`, etc.
- Parameter ID format changed: `processorId` → `nodeId` in fully qualified IDs
- JSON payload field names changed: `processorId` → `nodeId`

### Non-Breaking Changes
- Documentation terminology updated
- Architecture document headings updated
- Diagram labels updated
- Meta documentation updated

---

## H. Completion Status

✅ **All tasks completed successfully**

- [x] File renamed: `09-processors.md` → `09-nodes.md`
- [x] All architecture documents updated
- [x] All IPC specification files updated
- [x] Diagrams updated
- [x] Meta files updated
- [x] Cross-references validated
- [x] Remaining "processor" uses validated as acceptable (non-DSP context)

---

**Report Generated:** 2025-01-XX  
**Total Files Modified:** 17  
**Total Files Renamed:** 1

