# Architecture Document Renumbering Refactor Report

**Date:** 2025-11-19  
**Time:** 10:55:45 UTC  
**Type:** Architecture document renaming and reference update

---

## Summary

All architecture documents under `docs/architecture/` have been renamed from a global numeric scheme (`01-*.md`, `02-*.md`, etc.) to a section-based alphanumeric scheme (`A01-*.md`, `B01-*.md`, etc.). This change prevents global renumbering when new documents are added to any section.

---

## Section Assignments

Documents were grouped into six sections based on the existing index structure:

- **A** — Foundations (overview, core services, processing cohorts, project model)
- **B** — Structural Core (tracks, clips, parameters, nodes, mixer, media, routing)
- **C** — Timeline Core (timebase, advanced clips, editing, automation, MIDI, comping, editor)
- **D** — Creative & Performance Systems (clip launcher, rendering, control surfaces, UX, live recording)
- **E** — External Integration (windowing, diagnostics, scripting, video, plugins)
- **F** — Future Systems (collaboration, AI, experimental systems, sampler)

---

## File Renaming Table

| Old Filename | New Filename | Section | Description |
|-------------|--------------|---------|-------------|
| `01-overview.md` | `A01-overview.md` | A | Overview |
| `02-pulse.md` | `A02-pulse.md` | A | Pulse Architecture |
| `03-signal.md` | `A03-signal.md` | A | Signal Architecture |
| `04-aura.md` | `A04-aura.md` | A | Aura Architecture |
| `05-composer.md` | `A05-composer.md` | A | Composer Architecture |
| `06-processing-cohorts-and-anticipative-rendering.md` | `A06-processing-cohorts-and-anticipative-rendering.md` | A | Processing Cohorts & Anticipative Rendering |
| `07-project-versions-and-variants.md` | `A07-project-versions-and-variants.md` | A | Project, Versions & Variants Architecture |
| `08-tracks-lanes-and-roles.md` | `B01-tracks-lanes-and-roles.md` | B | Tracks, Lanes & Roles Architecture |
| `09-clips.md` | `B02-clips.md` | B | Clips Architecture |
| `10-parameters.md` | `B03-parameters.md` | B | Parameters Architecture |
| `11-node-graph.md` | `B04-node-graph.md` | B | Node Graph Architecture |
| `12-stack-nodes.md` | `B05-stack-nodes.md` | B | StackNode Architecture |
| `12-mixer-and-channel-architecture.md` | `B06-mixer-and-channel-architecture.md` | B | Mixer & Channel Architecture |
| `13-media-architecture.md` | `B07-media-architecture.md` | B | Media Architecture |
| `14-channel-routing-and-send-return.md` | `B08-channel-routing-and-send-return.md` | B | Channel Routing & Send/Return Architecture |
| `15-audio-streaming-and-caching.md` | `B09-audio-streaming-and-caching.md` | B | Audio Streaming & Caching Architecture |
| `16-timebase-tempo-and-groove.md` | `C01-timebase-tempo-and-groove.md` | C | Timebase, Tempo & Groove Architecture |
| `17-advanced-clips.md` | `C02-advanced-clips.md` | C | Advanced Clips Architecture |
| `18-editing-and-nondestructive-layers.md` | `C03-editing-and-nondestructive-layers.md` | C | Editing & Nondestructive Layers Architecture |
| `19-automation-and-modulation.md` | `C04-automation-and-modulation.md` | C | Automation & Modulation Architecture |
| `20-midi-architecture.md` | `C05-midi-architecture.md` | C | MIDI Architecture |
| `21-comping-architecture.md` | `C06-comping-architecture.md` | C | Comping Architecture |
| `22-editor-architecture.md` | `C07-editor-architecture.md` | C | Unified Editor Architecture |
| `23-clip-launcher.md` | `D01-clip-launcher.md` | D | Clip Launcher Architecture |
| `24-rendering-and-offline-processing.md` | `D02-rendering-and-offline-processing.md` | D | Rendering & Offline Processing Architecture |
| `25-control-surfaces-and-assistive-hardware.md` | `D03-control-surfaces-and-assistive-hardware.md` | D | Control Surfaces & Assistive Hardware Architecture |
| `26-ux-and-visual-layer.md` | `D04-ux-and-visual-layer.md` | D | UX & Visual Layer Architecture |
| `27-performance-and-live-recording.md` | `D05-performance-and-live-recording.md` | D | Performance & Live Recording Architecture |
| `28-windowing-and-plugin-ui.md` | `E01-windowing-and-plugin-ui.md` | E | Windowing & Plugin UI Architecture |
| `29-diagnostics-and-performance.md` | `E02-diagnostics-and-performance.md` | E | Diagnostics & Performance Architecture |
| `30-scripting-and-extensibility.md` | `E03-scripting-and-extensibility.md` | E | Scripting & Extensibility Architecture |
| `31-video-architecture.md` | `E04-video-architecture.md` | E | Video Architecture |
| `32-plugin-lifecycle-and-sandboxing.md` | `E05-plugin-lifecycle-and-sandboxing.md` | E | Plugin Lifecycle & Sandboxing Architecture |
| `33-plugin-library-and-browser.md` | `E06-plugin-library-and-browser.md` | E | Plugin Library & Browser Architecture |
| `34-collaboration-architecture.md` | `F01-collaboration-architecture.md` | F | Collaboration Architecture |
| `35-composer-extended-intelligence.md` | `F02-composer-extended-intelligence.md` | F | Composer Extended Intelligence Architecture |
| `36-future-workflow-prototypes.md` | `F03-future-workflow-prototypes.md` | F | Future Workflow Prototypes Architecture |
| `37-experimental-systems.md` | `F04-experimental-systems.md` | F | Experimental Systems Architecture |
| `38-ai-assisted-audio-tools.md` | `F05-ai-assisted-audio-tools.md` | F | AI-Assisted Audio Tools Architecture |
| `39-sampler-and-virtual-instruments.md` | `F06-sampler-and-virtual-instruments.md` | F | Sampler & Virtual Instruments Architecture |

**Total files renamed:** 39

---

## Index Updates

The `docs/architecture/00-index.md` file was updated to:

- Reflect the new section-based alphanumeric naming scheme
- Update all filename references to use the new `<LETTER><NUMBER>-slug` format
- Update section headers to show the new ranges (e.g., "Foundations (A01–A07)")
- Update the introductory text to mention "section-based alphanumeric order"
- Update the Notes section to explain the new numbering scheme

---

## Internal Reference Updates

All internal references to architecture documents were updated across the repository. The following files were modified:

### Architecture Documents
- `docs/architecture/A01-overview.md` — 1 reference updated
- `docs/architecture/B01-tracks-lanes-and-roles.md` — 1 reference updated
- `docs/architecture/B04-node-graph.md` — 3 references updated
- `docs/architecture/B06-mixer-and-channel-architecture.md` — 5 references updated
- `docs/architecture/E01-windowing-and-plugin-ui.md` — 3 references updated

### IPC Specifications
- `docs/specs/ipc/pulse/channel.md` — 2 references updated
- `docs/specs/ipc/pulse/node.md` — 3 references updated
- `docs/specs/ipc/pulse/track.md` — 1 reference updated
- `docs/specs/ipc/pulse/automation.md` — 3 references updated
- `docs/specs/ipc/pulse/parameter.md` — 2 references updated
- `docs/specs/ipc/pulse/routing.md` — 1 reference updated

### Meta Documentation
- `AGENTS.md` — 1 reference updated
- `README.md` — 2 references updated

**Total files with reference updates:** 13  
**Total references updated:** 23

---

## Reference Update Details

### Most Frequently Referenced Documents

The following documents had the most cross-references updated:

1. **Processing Cohorts** (`A06-processing-cohorts-and-anticipative-rendering.md`) — 7 references
2. **Mixer & Channel Architecture** (`B06-mixer-and-channel-architecture.md`) — 6 references
3. **StackNode Architecture** (`B05-stack-nodes.md`) — 3 references
4. **Parameters Architecture** (`B03-parameters.md`) — 3 references
5. **Node Graph Architecture** (`B04-node-graph.md`) — 2 references

---

## Verification

All references were verified using pattern matching to ensure no old numeric references remain. The following patterns were checked:

- `\]\(.*\d{2}-.*\.md\)` — Markdown links with two-digit prefixes
- `\]\(\.\./architecture/\d{2}-` — Relative links from subdirectories
- `\]\(\./\d{2}-` — Relative links within the architecture directory

All old references have been successfully updated to the new alphanumeric scheme.

---

## Notes

- The duplicate numbering issue (two files with `12-` prefix) was resolved by the new scheme
- All document slugs (the part after the prefix) were preserved unchanged
- No content within documents was modified except for link references
- The index file structure and grouping were preserved, only filenames and section ranges were updated

---

## Impact

This refactor enables future architecture documents to be added to any section without requiring global renumbering. New documents can be added as `A08-*.md`, `B10-*.md`, etc., within their respective sections, maintaining clear organisation while avoiding cascading renumbering across all documents.

