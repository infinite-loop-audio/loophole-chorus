
# Architecture Summary Report

This report provides a consolidated, objective summary of all architecture documents in the Loophole system, generated on 2025-11-19.

---

## 1. Overview of Architecture Coverage

All architecture documents scanned (in numerical order):

1. `00-index.md`
2. `01-overview.md`
3. `02-pulse.md`
4. `03-signal.md`
5. `04-aura.md`
6. `05-composer.md`
7. `06-processing-cohorts-and-anticipative-rendering.md`
8. `07-project-versions-and-variants.md`
9. `08-tracks-lanes-and-roles.md`
10. `09-clips.md`
11. `10-parameters.md`
12. `11-node-graph.md`
13. `12-mixer-and-channel-architecture.md`
14. `13-media-architecture.md`
15. `14-channel-routing-and-send-return.md`
15. `15-audio-streaming-and-caching.md`
16. `16-timebase-tempo-and-groove.md`
17. `17-advanced-clips.md`
18. `18-editing-and-nondestructive-layers.md`
19. `19-automation-and-modulation.md`
20. `20-midi-architecture.md`
21. `21-comping-architecture.md`
22. `22-editor-architecture.md`
23. `23-clip-launcher.md`
24. `24-rendering-and-offline-processing.md`
25. `25-control-surfaces-and-assistive-hardware.md`
26. `26-ux-and-visual-layer.md`
27. `27-performance-and-live-recording.md`
28. `28-windowing-and-plugin-ui.md`
29. `29-diagnostics-and-performance.md`
30. `30-scripting-and-extensibility.md`
31. `31-video-architecture.md`
32. `32-plugin-lifecycle-and-sandboxing.md`
33. `33-plugin-library-and-browser.md`
34. `34-collaboration-architecture.md`
35. `35-composer-extended-intelligence.md`
36. `36-future-workflow-prototypes.md`
37. `37-experimental-systems.md`
38. `38-ai-assisted-audio-tools.md`
39. `39-sampler-and-virtual-instruments.md`

**Total: 40 architecture documents**

---

## 2. High-Level System Overview

Loophole is a Digital Audio Workstation (DAW) ecosystem built on a multi-process architecture with clear separation of concerns between real-time audio processing and non-real-time project management.

### System Components

The system consists of five repositories:

- **Signal** (C++, JUCE): Real-time audio engine responsible for plugin hosting, audio graph execution, DSP processing, and hardware I/O. Must remain real-time safe and deterministic.

- **Pulse** (TypeScript): Authoritative project and data model layer. Owns all project state including tracks, clips, channels, routing, automation, and parameters. Constructs engine graphs for Signal and coordinates with Composer for metadata.

- **Aura** (Electron + TypeScript): User interface layer providing all visual representation, editing workflows, and user interaction. Does not store authoritative state; mirrors Pulse and sends editing intents.

- **Composer** (TBD): External knowledge metadata service providing shared plugin and parameter metadata, semantic roles, mapping suggestions, and deterministic behaviour classification. Optional and advisory.

- **Chorus**: Meta repository containing architecture documents, specifications, ADRs, IPC schemas, and meta-protocol definitions. No runtime code.

### Key Architectural Principles

1. **Clear Separation of Concerns**: Real-time (Signal) and non-real-time (Pulse, Aura) responsibilities never mix. This protects audio stability.

2. **Multi-Process Model**: Major responsibilities isolated into separate processes to prevent plugin crashes from affecting UI and allow independent scaling.

3. **Declarative Contracts**: Communication between layers uses explicit schemas defined in IPC specifications. No implicit behaviour permitted.

4. **Replaceability**: Any layer can be replaced independently without affecting others.

### Major Conceptual Layers

1. **Project Layer**: Projects contain Arrangements, Versions, and Variants. Arrangements hold timeline structures (Tracks, Clips, Lanes).

2. **Track/Channel Layer**: Tracks are user-facing containers for Clips. Channels are engine-level signal processors with NodeGraphs. A Track may optionally have a Channel.

3. **Clip/Lane Layer**: Clips are time-bounded containers on Tracks containing one or more Lanes (audio, MIDI, automation, video). Lanes route to Channels via LaneStreamNodes.

4. **Node Graph Layer**: Linear processing chains within Channels containing instruments, effects, utility nodes, and LaneStreamNodes. Pulse constructs graphs; Signal executes them.

5. **Processing Cohort Layer**: Dual-engine architecture separating nodes into live (real-time, low-latency) and anticipative (pre-rendered, large buffers) cohorts for scalability.

6. **Parameter/Automation Layer**: Unified parameter model with automation lanes, modulation sources, and sample-accurate envelope evaluation.

7. **Media Layer**: First-class media assets with project and global library scopes, robust handling of offline/remote storage, and analysis artefacts.

8. **Metadata/Intelligence Layer**: Composer provides plugin and parameter intelligence, semantic mapping, and deterministic behaviour classification.

### Guiding Architectural Principles

- **Pulse is authoritative**: All project state lives in Pulse. Aura mirrors; Signal executes.
- **Determinism first**: Projects must be deterministic and reproducible. AI and generative features must store explicit results.
- **Non-destructive by default**: Editing uses pipelines and layers. Ghost renders and comping preserve original material.
- **IPC-based communication**: All inter-process communication uses explicit, versioned IPC contracts.
- **Real-time safety**: Signal must never block, allocate, or perform non-deterministic operations in audio threads.
- **Cohort-aware processing**: Nodes are dynamically assigned to live or anticipative cohorts based on determinism, routing, and user interaction.

---

## 3. Document-by-Document Structured Summaries

### `00-index.md`

#### Purpose
Provides a numerical index of all architecture documents, organised by thematic sections (Foundations, Structural Core, Timeline Core, Creative & Performance Systems, External Integration, Future Systems).

#### Key Concepts
- Filename numbering is authoritative
- Documents grouped into logical sections
- Notes on planned and future items

#### Responsibilities / Components
- Index organisation
- Document categorisation

#### Inputs / Outputs / Dependencies
- None (meta-document)

#### Notes & Constraints
- Filename numbering must be maintained as architecture expands
- Items marked (planned) or (future) do not constrain v1

---

### `01-overview.md`

#### Purpose
Primary architectural overview defining conceptual responsibilities of each subsystem, runtime boundaries, and the multi-process model.

#### Key Concepts
- Multi-process architecture (Signal, Pulse, Aura, Composer, Chorus)
- Clear separation of RT and NRT concerns
- Declarative contracts via IPC
- Processing cohorts (live and anticipative)
- Replaceability of layers

#### Responsibilities / Components
- **Signal**: Real-time engine, plugin hosting, audio graph execution, hardware I/O
- **Pulse**: Project state authority, graph construction, parameter management, cohort assignment
- **Aura**: UI layer, visualisation, user interaction, plugin window embedding
- **Composer**: Shared metadata service, parameter mapping, deterministic classification
- **Chorus**: Meta repository, specifications, IPC schemas

#### Inputs / Outputs / Dependencies
- Pulse → Signal: Graph instructions, parameter updates, automation streams
- Pulse → Aura: State snapshots, validation results
- Pulse → Composer: Metadata queries (optional, advisory)
- Aura → Pulse: Editing intents
- Composer accessed via network APIs (HTTP/HTTPS)

#### Notes & Constraints
- Normative language (MUST/SHOULD/MAY) used throughout
- Composer MUST NOT be required for core project loading
- Signal MUST NOT store project structures
- Aura MUST NOT talk directly to Signal or Composer
- British English spelling convention

---

### `02-pulse.md`

#### Purpose
Defines Pulse as the authoritative data and sequencing layer hosting the full project model, resolving editing operations, constructing engine graphs, and coordinating with other components.

#### Key Concepts
- Project state authority
- Transactional editing model
- Channel and node graph construction
- Parameter identity, aliasing, and automation
- Composer coordination (advisory)
- Processing cohort assignment
- Hardware device registry and mapping

#### Responsibilities / Components
- Project state ownership (Tracks, Clips, Channels, Lanes)
- Editing and validation (transactional, undo/redo)
- Channel and node graph construction for Signal
- Parameter identity, mapping, and automation resolution
- Composer integration (metadata acquisition, local caching)
- Incremental state change emission to Aura and Signal
- Persistence (project files, drafts, deterministic serialisation)
- Processing cohort assignment (live vs anticipative)
- Hardware device registry, mapping engine, and feedback system

#### Inputs / Outputs / Dependencies
- **To Signal**: Structural messages (channel/node graph updates), parameter/automation streams, gesture streams, graph rebase instructions, feedback intents for hardware
- **To Aura**: State projections (snapshots), validation results, editing APIs
- **From Composer**: Plugin metadata, parameter metadata, semantic roles, mapping suggestions, deterministic behaviour metadata, device profiles
- **From Signal**: Control messages (hardware inputs), engine status events

#### Notes & Constraints
- Pulse MUST ensure project state remains valid without Composer
- Pulse owns all cohort assignment decisions
- Editing operations are atomic transactions with validation
- Graph updates to Signal are incremental and real-time safe
- Composer data is advisory; Pulse maintains authority

---

### `03-signal.md`

#### Purpose
Defines Signal as the real-time audio engine responsible for deterministic audio processing, plugin hosting, parameter application, and transport management.

#### Key Concepts
- Real-time safety constraints
- Dual-engine architecture (live and anticipative cohorts)
- Plugin hosting (VST3, CLAP, AU)
- Parameter application (automation and gestures)
- Hardware I/O and control surfaces (low-level transport)
- Render horizon for anticipative processing

#### Responsibilities / Components
- Plugin hosting (loading, initialising, running instruments and effects)
- Real-time DSP execution
- Parameter updates (automation and gestures at audio rate)
- Transport (playback position, tempo, sync, sample clocks)
- Audio I/O (capturing and rendering to hardware)
- Engine graph execution (linear node graph from Pulse)
- Hardware device enumeration and I/O (MIDI, HID, OSC)
- Processing cohorts (live engine in audio thread, anticipative engine on worker threads)

#### Inputs / Outputs / Dependencies
- **From Pulse**: Structural commands (graph updates), parameter/automation commands, gesture streams, transport state
- **To Pulse**: Engine status events (plugin failures, processing errors, transport confirmations), control messages (hardware inputs), feedback for hardware outputs
- **From Hardware**: Raw inputs (MIDI, HID, OSC)
- **To Hardware**: Feedback outputs (LEDs, displays, meters)
- Signal has zero awareness of Composer

#### Notes & Constraints
- Signal MUST NOT perform dynamic allocation in audio thread
- Signal MUST NOT block or allocate in RT code
- Signal NEVER interacts with Composer
- Graph updates must be committed outside audio thread and swapped atomically
- Anticipative engine maintains render horizon ahead of playhead
- Cohort transitions must be handled smoothly with crossfading

---

### `04-aura.md`

#### Purpose
Defines Aura as the user interface layer responsible for all visual representation, user interaction, editing workflows, and intent generation.

#### Key Concepts
- No authoritative state (mirrors Pulse)
- MVC relationship (Pulse = Model, Aura = View + Controller)
- Ephemeral UI state only
- Intent-based editing
- Plugin UI embedding (windows from Signal)
- Context signals for hardware mapping

#### Responsibilities / Components
- Rendering (visualising Tracks, Clips, Lanes, Channels, nodes, automation, parameters)
- User interaction (mouse, keyboard, touch, gestures)
- Intent generation (converting interactions into editing commands for Pulse)
- Plugin UI embedding (hosting plugin windows from Signal)
- Feedback presentation (transport state, metering, analysis, warnings, errors)
- Context signalling (active view, focus state for hardware mapping)

#### Inputs / Outputs / Dependencies
- **To Pulse**: Editing intents, context signals (active view, focus state)
- **From Pulse**: Snapshot events (project state projections), validation results, error messages
- **From Signal**: Plugin window handles, metering/analysis data (via side channel)
- Aura NEVER communicates with Composer directly
- Aura NEVER talks directly to Signal for project data

#### Notes & Constraints
- Aura MUST NOT store project-critical data
- Aura MUST NOT interpret routing or construct audio graphs
- Aura MUST NOT bypass Pulse for structural data
- Deterministic rendering based on Pulse snapshots
- Visual virtualisation required for performance
- High-frequency updates must be throttled and batched

---

### `05-composer.md`

#### Purpose
Defines Composer as Loophole's shared knowledge metadata service providing plugin and parameter metadata, semantic roles, mappings, and deterministic behaviour classification.

#### Key Concepts
- External optional service
- Plugin identity normalisation (family and variant model)
- Parameter metadata and semantic roles
- Mapping suggestions (version updates, format switches, plugin replacements)
- Hardware device intelligence and profiles
- Deterministic behaviour telemetry and inference
- Privacy-first (opt-in, anonymised)

#### Responsibilities / Components
- Plugin metadata (identity normalisation, parameter metadata, semantic roles)
- Parameter identity (variant-level and family-level IDs)
- Semantic roles (classifying parameters by meaning for cross-plugin mapping)
- Plugin categories and tags
- Mapping suggestions (version/format/plugin transitions)
- Hardware device metadata (identity, capability fingerprints, default mappings)
- Control surface device intelligence (device profiles, default mappings)
- Deterministic behaviour telemetry and inference

#### Inputs / Outputs / Dependencies
- **From Pulse**: Plugin identity queries, parameter metadata requests, device fingerprints, telemetry (opt-in, anonymised)
- **To Pulse**: Plugin metadata, parameter metadata, semantic roles, mapping suggestions, deterministic classification, device profiles
- Composer accessed via network APIs (HTTP/HTTPS)
- Composer MUST NOT be required for core functionality
- Composer MUST NOT receive audio, arrangement data, or personally identifiable information

#### Notes & Constraints
- Composer is optional and non-blocking
- All suggestions are advisory; Pulse maintains authority
- Privacy boundaries: no audio, no project content, no identifiers, only aggregated behavioural signatures
- Local caching required for offline operation
- Composer contributions are opt-in only

---

### `06-processing-cohorts-and-anticipative-rendering.md`

#### Purpose
Defines the dual-domain audio processing system (processing cohorts) enabling simultaneous low-latency live performance and heavy DSP anticipative rendering.

#### Key Concepts
- Live cohort: real-time, low-latency (64-128 samples), for live input and non-deterministic nodes
- Anticipative cohort: pre-rendered, large buffers (hundreds of ms to seconds), for deterministic nodes
- Cohort assignment by Pulse based on liveness, determinism, routing dependencies
- Render horizon: anticipative buffer ahead of playhead
- Dynamic cohort transitions with crossfading

#### Responsibilities / Components
- **Pulse**: Cohort assignment decisions, graph analysis, dependency closure, user overrides
- **Signal**: Dual-engine execution (real-time engine for live, anticipative engine for pre-render), render horizon management, cohort transitions

#### Inputs / Outputs / Dependencies
- **Pulse → Signal**: Cohort assignment properties per node, cohort update messages
- **Composer → Pulse**: Deterministic behaviour metadata to inform cohort assignment
- **Aura → Pulse**: Context (open plugin UIs, armed tracks) affecting liveness
- Nodes marked live if: live audio/MIDI input, open GUI, non-deterministic, downstream of live nodes

#### Notes & Constraints
- Pulse owns all cohort membership decisions
- Live nodes always upstream; anticipative nodes downstream only if safe
- Render horizon must stay ahead of playhead
- Cohort transitions must be smooth with crossfading
- User overrides supported (Force Live, Prefer Pre-Render)

---

### `07-project-versions-and-variants.md`

#### Purpose
Defines the project-level model including Projects, Arrangements, Versions, and Variants, plus the two-step persistence model (draft vs save).

#### Key Concepts
- Project: root model entity with arrangements, metadata, versions
- Arrangement: timeline configuration of tracks, clips, automation, nodes
- Version: structural checkpoint (project-level or arrangement-level)
- Variant: lightweight alternative of sub-structures (track, clip, mix variants)
- Two-step persistence: draft saves (automatic) vs explicit saves (commit)

#### Responsibilities / Components
- Project lifecycle (loading, saving, drafting)
- Arrangement management (multiple arrangements per project)
- Version control (immutable snapshots, branching, reverting)
- Variant management (cheap to create, easy to switch)
- Persistence (project files, draft files, deterministic serialisation)

#### Inputs / Outputs / Dependencies
- **Pulse → Aura**: Project snapshots, version metadata
- **Aura → Pulse**: Version/variant creation commands
- Undo/redo operates within current working tree; versions represent structural checkpoints

#### Notes & Constraints
- Only one active project per Pulse instance
- Draft saves never silently override main project file
- Versions are immutable snapshots; current state is mutable working tree
- Variants are local and scoped to arrangements
- Version revert is itself undoable

---

### `08-tracks-lanes-and-roles.md`

#### Purpose
Defines the conceptual structure of Tracks, Channels, and Lanes, maintaining strict separation between editorial (Tracks), signal processing (Channels), and content (Lanes).

#### Key Concepts
- Unified Track model: one Track type capable of hosting Clips, nested Tracks, and optionally a Channel
- Channels: engine-level signal processors with NodeGraphs
- Lanes: typed content streams inside Clips (audio, MIDI, automation, video)
- LaneStreamNodes: bridge Lanes to Channel nodes
- Track nesting for structural organisation

#### Responsibilities / Components
- **Tracks**: User-facing containers for Clips, editing interfaces, timeline structure
- **Channels**: Audio signal processing (nodes, routing, parameters)
- **Lanes**: Content definition (audio, MIDI, automation, video)
- **LaneStreamNodes**: First-class nodes receiving audio from Clip Lanes

#### Inputs / Outputs / Dependencies
- **Clips → Lanes**: Clips contain and own Lanes
- **Lanes → Channels**: Audio Lanes route to LaneStreamNodes; MIDI Lanes route to instruments
- **Composer → Pulse**: Metadata influences interpretation but does not alter architecture
- **Signal**: Executes Channel node graphs

#### Notes & Constraints
- Tracks define editing; Channels define audio processing; Lanes define content
- Tracks do not directly own Lanes; Clips own Lanes
- A Track may have zero or one Channel
- LaneStreamNodes created lazily on demand
- Global Lanes (tempo, groove) exist outside Tracks

---

### `09-clips.md`

#### Purpose
Defines the conceptual model for Clips as time-bounded containers on Tracks, their relationship to Lanes, and how Pulse resolves Clip contents during playback.

#### Key Concepts
- Clip: time-bounded container on a Track with one or more Lanes
- Clip time range (start, end, length) in musical time
- Per-Clip lane routing (audio to LaneStreamNodes, MIDI to instruments)
- Clip editing rules (split, consolidate, overlap handling)

#### Responsibilities / Components
- Clip placement and timing (time ranges, snap, grid, stretch)
- Clip identity (stable IDs, editing state, transient IDs)
- Lanes inside Clips (ordered, typed collections)
- Content in Lanes (audio, MIDI, automation, video)
- Playback resolution (Pulse determines active Clips and routes Lanes)

#### Inputs / Outputs / Dependencies
- **Tracks**: Clips are placed on Tracks
- **Lanes**: Clips contain and own Lanes
- **Channels**: Clip Lanes route to Channel nodes
- **Composer**: Provides metadata for automation mapping (advisory)

#### Notes & Constraints
- Clips define strict time ranges (start < end)
- Different Clips on a Track can contain different lane configurations
- Overlapping Clips: later Clip takes precedence
- Clip boundaries do not automatically imply fades or truncation
- Time ranges use project time units (beats or seconds depending on timebase)

---

### `10-parameters.md`

#### Purpose
Defines how parameters are represented, identified, mapped, automated, and resolved across the system, supporting stability across plugin versions and formats.

#### Key Concepts
- Fully qualified parameter IDs (entityType.entityId.nodeId.parameterId)
- Logical vs variant parameters (family-level stability vs format-specific)
- Parameter aliasing (preserving automation across plugin changes)
- Semantic roles (meaning-based parameter classification)
- Automation binding (parameter IDs to automation lanes)

#### Responsibilities / Components
- Parameter identity management (stable IDs across saves, undo/redo, plugin changes)
- Parameter types (continuous, discrete, compound, hidden)
- Parameter aliasing (local alias table, Composer metadata)
- Semantic roles (meaning-based classification for cross-plugin mapping)
- Automation resolution (binding, data storage, gesture vs playback streams)

#### Inputs / Outputs / Dependencies
- **Pulse**: Maintains authoritative parameter state, alias tables, automation binding
- **Signal**: Receives resolved parameter updates, applies at audio rate
- **Aura**: Displays and edits parameters via Pulse
- **Composer**: Provides parameter metadata, semantic roles, mapping suggestions (advisory)

#### Notes & Constraints
- Parameter IDs MUST remain stable across saves, undo/redo, plugin format switches
- Pulse manages stability using alias tables and Composer metadata
- Signal treats parameters as numeric identifiers and values only
- Automation stored at Clip scope but resolved at playback time
- Composer is optional; Pulse falls back to local aliasing

---

### `11-node-graph.md`

#### Purpose
Defines the conceptual model for nodes in Loophole, how they form Channel chains, and their relationship to Tracks, Clips, and Lanes.

#### Key Concepts
- Nodes: fundamental DSP processing elements inside Channels
- Serial node chains (linear, strictly ordered)
- Node types: instruments, audio effects, LaneStreamNodes, utility, device/IO
- Node identity (stable IDs, plugin identity vs node identity)
- Node lifecycle (creation, replacement, removal, bypassing)

#### Responsibilities / Components
- **Node Types**: Instruments (MIDI → audio), audio effects (audio → audio), LaneStreamNodes (audio from Lanes), utility (gain, pan, meters), device/IO
- **Node Graph**: Linear serial chains within Channels
- **Parameter Ownership**: Nodes own parameters
- **Node Lifecycle**: Creation, replacement (with parameter remapping), removal, bypassing

#### Inputs / Outputs / Dependencies
- **Pulse**: Manages node graph model, creates/updates/removes nodes, assigns node IDs, uses Composer for plugin semantics
- **Signal**: Instantiates nodes, manages plugin lifecycles, applies parameters, executes graphs
- **Aura**: Displays node graphs, allows editing, opens plugin UIs
- **Composer**: Provides plugin identity normalisation, parameter metadata, semantic roles (affects Pulse's interpretation)

#### Notes & Constraints
- Node graphs are strictly serial (linear chains)
- Nodes MUST NOT own project-level state
- LaneStreamNodes are first-class nodes
- Node IDs stable across Track duplication, simple reordering, parameter edits
- Pulse ensures automation bindings updated if node IDs change
- Signal never interprets semantic roles or aliasing

---

### `12-mixer-and-channel-architecture.md`

#### Purpose
Describes Loophole's mixing model and console architecture, where everything in the signal path is a Node (faders, pans, sends as nodes).

#### Key Concepts
- Faders as Nodes (FaderNode)
- Sends as Nodes (SendNode)
- Pre/post behaviour via Node topology (SendNode position relative to FaderNode)
- Control Groups (VCA-like behaviour as parameter groups, not separate Channel types)
- No separate "inserts" and "sends" entities

#### Responsibilities / Components
- **FaderNode**: Controls final gain for Channel
- **PanNode**: Implements panning/balance
- **SendNode**: Taps signal and routes to target Channel
- **Control Groups**: Parameter groups for VCA-style behaviour
- Console UI (default mixer view and advanced graph view)

#### Inputs / Outputs / Dependencies
- **Channels**: Contain node graphs with FaderNodes, PanNodes, SendNodes
- **Routing**: SendNodes reference routing graph
- **Parameters**: Control groups defined in Parameter domain
- **Processing Cohorts**: Nodes assigned to cohorts affect mixer processing

#### Notes & Constraints
- All Channels use same FaderNode concept regardless of role
- Pre/post fader behaviour is positional (SendNode before/after FaderNode)
- VCA-style behaviour implemented via parameter grouping, not Channel types
- Folder tracks may have Channels (folder-as-bus scenarios)
- MIDI tracks do not have separate "MIDI faders" in mixer

---

### `13-media-architecture.md`

#### Purpose
Defines architecture for media management, browsing, discovery, and handling of offline/remote storage in Loophole.

#### Key Concepts
- Media Items: canonical representation of media assets (audio, MIDI, video, analysis)
- Storage backends (project-local, cloud-backed, external drives)
- Availability status model (lifecycle + physical availability)
- Project Library vs Global Library
- Analysis artefacts (waveform overviews, loudness envelopes, beat grids, spectral summaries, embeddings)
- Kits and Pad Workspaces

#### Responsibilities / Components
- **Media Items**: Stable per-project identifiers, storage descriptors, format metadata, status, tags
- **Storage Backends**: Project-local, cloud-backed filesystems, external volumes, external references
- **Analysis Artefacts**: Derived data products (waveforms, envelopes, beat grids, spectral summaries, embeddings)
- **Media Browser**: Multiple browsing modes (list, waveform thumbnails, visual cluster, hypergrid, folder/pack, project view)
- **Kits**: Structured collections of media or instrument presets (drum kits, vocal kits, texture kits)
- **Pad Workspaces**: Interactive environments for kit experimentation

#### Inputs / Outputs / Dependencies
- **Pulse**: Owns media metadata, availability status, storage descriptors
- **Signal**: Performs heavy audio analysis, provides analysis artefacts
- **Aura**: Provides media browser UI, audition/preview
- **Composer**: Augments search with semantic similarity, provides tags and categorisation

#### Notes & Constraints
- Media availability MUST never be assumed
- All media I/O operations asynchronous and cancellable
- Missing media never "forgotten" unless user explicitly removes
- Project integrity preserved even when media offline
- Cloud-backed filesystems treated as untrusted for latency
- Analysis artefacts stored as sidecar objects or inline blobs

---

### `14-channel-routing-and-send-return.md`

#### Purpose
Defines the channel routing model with focus on sends, returns, and signal flow between Channels, treating sends and inserts as the same kind of building block.

#### Key Concepts
- Sends as Nodes (SendNode in source Channel's NodeGraph)
- Returns as Channels (destination Channels receiving send inputs)
- Routing graph maintained by Pulse (project-level connections)
- Sidechain routing (source Channel to Node's sidechain input)
- Hardware routing (Channels to hardware I/O)

#### Responsibilities / Components
- **Routing Graph**: Project-level graph connecting Channels (main routes, sends, sidechains, hardware)
- **SendNodes**: Nodes in source Channel that tap signal and route to target
- **Return Channels**: Destination Channels receiving send inputs
- **Sidechain Routes**: Specialised routes to Node sidechain inputs
- **Hardware Routes**: Channel to hardware I/O mapping

#### Inputs / Outputs / Dependencies
- **Pulse**: Maintains routing graph, validates topology, coordinates with Channel/Node domains
- **Signal**: Instantiates connections between Channel processes based on routing graph
- **Routing & Cohorts**: Some routing patterns constrain anticipative rendering
- Pre/post behaviour determined by SendNode position in Node graph

#### Notes & Constraints
- Routing is explicit and inspectable (no hidden special-case behaviour)
- SendNodes tap signal at their position in Node list
- Multiple SendNodes can target same destination (summing handled automatically)
- Routing graph enforces no infinite cycles (safety constraints)
- Routing changes are non-realtime operations (may trigger graph rebuilds)

---

### `15-audio-streaming-and-caching.md`

#### Purpose
Defines architecture for audio file streaming, caching, prefetching, and disk I/O, integrated with anticipative rendering.

#### Key Concepts
- Two-tier streaming model: Disk Stream Buffers (low-level) and Clip-Window Buffers (high-level)
- Anticipative-aware prefetching (Pulse requests future sections)
- Three-level cache model (hot buffer, warm clip cache, cold global asset cache)
- Proxy assets (lower resolution for editing)
- Waveform and peak cache (precomputed overviews)

#### Responsibilities / Components
- **Signal**: Handles all audio streaming, disk I/O, decode operations, buffer management
- **Pulse**: Manages asset metadata, caching policies, anticipative scheduling (ClipWindow requests)
- **Aura**: Displays status, thumbnails, waveform previews

#### Inputs / Outputs / Dependencies
- **Pulse → Signal**: Asset registration, ClipWindow requests, stream start/stop commands
- **Signal → Pulse**: Underrun events, asset status changes, prefetch completion
- **Signal**: Disk streaming worker threads, Stream Buffers, decode pipelines
- **Anticipative Rendering**: Streaming subsystem must pre-read required audio

#### Notes & Constraints
- Signal handles all audio streaming; Pulse never touches raw sample data
- All streaming asynchronous and non-blocking
- Proxy assets used when playback resolution or system load requires
- Waveform previews never decoded during playback (precomputed)
- Missing/cloud files: Signal does not block, streams proxy or silence, notifies Pulse

---

### `16-timebase-tempo-and-groove.md`

#### Purpose
Defines complete timebase, tempo, and groove architecture underpinning all timeline-driven subsystems.

#### Key Concepts
- Three-layered time system: Absolute Time (seconds), Sample Time (samples), Musical Time (bars/beats/ticks)
- Timebase: global musical timing context (TPQN, bar length, beat unit)
- Tempo Map: BPM over time (step changes, linear ramps, exponential ramps)
- Groove System: timing offsets applied to notes, audio slices, automation
- Sample-accurate mapping between all time domains

#### Responsibilities / Components
- **Timebase**: Global musical timing context with time signature map
- **Tempo Map**: List of tempo events (position, BPM, curve type)
- **Groove System**: Groove templates with per-step offsets, velocity offsets
- **Conversions**: Pulse computes all time domain conversions (beats ↔ samples, musical ↔ samples)

#### Inputs / Outputs / Dependencies
- **Pulse**: Maintains all timebase-related state, converts all musical-time to sample-time
- **Signal**: Uses sample-domain timing exclusively
- **Clips**: Depend on tempo for warp maps, stretch factors
- **MIDI**: Depends on beat position, groove application, quantisation
- **Automation**: Points stored in musical time, converted to samples
- **Rendering**: Requires sample-aligned tempo map segments

#### Notes & Constraints
- All MIDI events stored in musical time
- Tempo ramps must be sample-accurate
- Groove applied in musical time before conversion to samples
- Signal never calculates tempo curves; receives resolved sample-based instructions
- Pulse provides deterministic conversion API

---

### `17-advanced-clips.md`

#### Purpose
Expands foundational clip architecture with advanced features: warp & stretch, multi-lane capabilities, slicing & transient detection, clip variants, clip-level DSP, containers, and ARA integration.

#### Key Concepts
- Multi-lane clips (audio, MIDI, automation, video, device lanes)
- Warp maps (audio-time to musical-time mapping with WarpPoints)
- Transient maps (detection markers for slicing)
- Clip variants (lightweight alternatives with different edits)
- Clip containers (nested clips, group clips)
- Clip-level DSP (processing before Track-level)
- ARA integration (clip-scoped external editors)

#### Responsibilities / Components
- **Clip Structure**: Multi-lane content, warp maps, transient maps, quantisation, groove, variants, clip DSP chain
- **Audio Clip Model**: Dual representation (source-time samples, musical-time beats), warp maps, stretch algorithms
- **MIDI Clip Model**: Notes, expression, MPE, editing layers (quantise, groove, randomisation, etc.)
- **Clip Variants**: Different edits of same conceptual clip
- **Clip Containers**: Clips containing child Clips
- **ARA Regions**: Clip-scoped editing regions for ARA-capable plugins

#### Inputs / Outputs / Dependencies
- **Pulse**: Owns clip model, resolves warp maps, transient maps, variants
- **Signal**: Executes audio playback, receives sample-accurate warp instructions
- **ARA Plugins**: Operate on clip audio lanes, provide deep musical-time editing
- **Editing Tools**: Warp editing, transient alignment, slicing modes

#### Notes & Constraints
- WarpPoints define mapping between audio samples and musical beats
- Ghost MIDI used for performance when pipelines complex
- Clip variants scoped to clips (not Track/Arrangement level)
- ARA regions attach to stable clip audio (not raw takes)
- Clip containers allow reusable riffs and multi-track patterns

---

### `18-editing-and-nondestructive-layers.md`

#### Purpose
Defines the editing model and nondestructive layer stack for audio and MIDI, using clip edit pipelines and ghost renders.

#### Key Concepts
- Layered editing model (Media Level, Clip Level, Clip DSP Chain, Track/Node Graph)
- Clip Edit Pipelines: stacked transforms (hostOps, nodeOps, araOps)
- Ghost Renders: materialised derived assets for heavy pipelines
- Media Editor: creates derived assets rather than overwriting
- MIDI Edit Pipelines: stacked transforms for MIDI content
- Media Archiving: compression of obsolete assets (FLAC)

#### Responsibilities / Components
- **Clip Edit Pipelines**: Stack of edit steps (host-native, Node-based, ARA-based)
- **Ghost Renders**: Derived media assets created when pipelines are materialised
- **Media Editor**: Uses same primitives but produces new derived assets
- **MIDI Edit Pipelines**: Transform layers (quantise, humanise, scale snap, etc.)
- **Media Archiving**: Obsolete assets compressed and archived

#### Inputs / Outputs / Dependencies
- **Pulse**: Decides when to ghost render vs live evaluation, manages pipelines and derived assets
- **Signal**: Executes Node-based DSP and ARA, provides preview/processing
- **ARA**: Integrated as pipeline steps
- **Editing Tools**: Operations expressed as pipeline steps

#### Notes & Constraints
- Editing never silently destroys original material
- Ghost renders stored as normal media assets with lineage
- Clip-Safe Nodes defined (subset of Nodes allowed in pipelines)
- Media-level edits create new derived assets, not in-place overwrites
- Pulse decides evaluation mode (live vs ghost render) based on cost

---

### `19-automation-and-modulation.md`

#### Purpose
Defines complete automation and modulation model covering automation lanes, parameter envelopes, modulation sources, routing, and sample-accurate evaluation.

#### Key Concepts
- Automation domains: Clip, Track, Lane, Node levels
- Parameter binding via fully qualified IDs
- Automation lanes with envelopes (points with shapes: step, linear, curve, bezier)
- Modulation system: LFOs, envelope followers, MIDI-based, sidechain, random sources
- Evaluation model: musical-time automation converted to sample-domain, modulators as sample streams
- Automation comping: take lanes and comp layers for automation

#### Responsibilities / Components
- **Automation Lanes**: Contain envelopes targeting parameters via IDs
- **Modulation System**: Modulators (LFOs, envelope followers, MIDI-based) routing to parameters
- **Evaluation Pipeline**: Pulse resolves automation to sample-accurate envelopes; Signal executes
- **Automation Comping**: Take lanes and comp layers for automation curves

#### Inputs / Outputs / Dependencies
- **Pulse**: Owns automation graph, resolves musical-time to samples, manages modulation routing
- **Signal**: Executes sample-accurate automation and modulation streams
- **Clips**: Contain automation lanes
- **Nodes**: Expose parameters for automation/modulation
- **Timebase**: Automation points stored in musical time, converted using tempo/warp/groove

#### Notes & Constraints
- Automation MUST be sample-accurate and deterministic
- Automation stored in musical time, evaluated in sample space
- Multiple modulators can target same parameter (summation rules)
- Automation comping works like audio comping (take lanes, comp layers)
- Pulse is authority for resolving musical-time automation; Signal executes sample-domain

---

### `20-midi-architecture.md`

#### Purpose
Defines full MIDI and note-expression architecture including MPE, per-note expression, clip-level and media-level MIDI, and integration with editing pipelines.

#### Key Concepts
- Structured MIDI event model preserving musical-time positions and per-note expression
- MPE compatibility (per-note channel assignment, independent pitch/timbre/pressure)
- MIDI in Clips: MIDI lanes with base notes and edit pipelines
- MIDI Edit Pipeline: nondestructive transforms (quantise, humanise, scale snap, etc.)
- Ghost MIDI: flattened note list for performance when pipelines complex
- Groove integration: applies to note onsets, end times, CC ramp alignment

#### Responsibilities / Components
- **MIDI Representation**: Note events with expression curves, CC events, pitch bend, channel pressure, polyphonic aftertouch
- **MIDI Lanes**: Base notes, edit pipelines, ghost MIDI
- **MIDI Edit Pipeline**: Stackable, nondestructive transform layers
- **Groove Integration**: Applies at clip, track, arrangement levels
- **MIDI Nodes**: MIDI-capable nodes in NodeGraph (filters, generators, LFOs, arpeggiators)

#### Inputs / Outputs / Dependencies
- **Pulse**: Owns canonical MIDI data, resolves to sample-accurate scheduling
- **Signal**: Performs sample-accurate scheduling and modulation
- **Clips**: Contain MIDI lanes
- **Instruments**: Receive MIDI from lanes
- **Timebase**: MIDI events stored in musical time, converted using tempo/warp/groove

#### Notes & Constraints
- All MIDI events stored in musical time
- Ghost MIDI materialised when pipelines complex or for stability
- MPE envelopes stored as per-note automation
- MIDI editing is nondestructive by default (pipelines, not direct edits)
- Signal receives flattened MIDI data for deterministic execution

---

### `21-comping-architecture.md`

#### Purpose
Defines full comping architecture across audio, MIDI, and automation domains using nondestructive layered model with take lanes and comp layers.

#### Key Concepts
- Take lanes: recording produces take lanes (never overwriting)
- Comp layers: swipe comping selects segments from takes
- Result clips: compiled results for playback
- Multi-track comp groups: phase-locked comping across tracks
- ARA integration: only applies to compiled clips, invalidated when comping changes
- Nondestructive: takes and comp layers remain available for revision

#### Responsibilities / Components
- **Take Lanes**: Audio, MIDI, automation take lanes created during recording
- **Comp Layers**: Sets of comp selections assembled from takes (with segments)
- **Result Clips**: Active comp layer produces result clip in Track's main lane
- **Multi-Track Comp Groups**: Linked tracks with phase-locked or free-sync comping
- **Compilation**: Pulse generates result clips when comp edits change

#### Inputs / Outputs / Dependencies
- **Recording**: Produces take lanes automatically
- **Compiling**: Produces result clips feeding ClipEditPipeline
- **ARA**: Only applies to compiled clips; invalidated when comping changes
- **Editing Pipelines**: Result clips become input to ClipEditPipelines
- **Rendering**: Pulse resolves compiled segments, crossfades, warp maps

#### Notes & Constraints
- Recording always produces new take lanes (never destructive)
- Comp layers fully editable and reversible
- ARA regions invalidated when comping modified upstream
- Multi-track groups support phase-aligned comping
- Comp boundaries align with musical time or transient markers
- Result clips participate in full editing pipeline (ARA, clip DSP, etc.)

---

### `22-editor-architecture.md`

#### Purpose
Defines unified editing platform for all Loophole editors with shared core model, tools, selection, snapping, transactions, and gesture integration.

#### Key Concepts
- Editor Core Model: shared state (view state, selection, tools, snapping, focus)
- Editing primitives: atomic edit instructions (time operations, value operations, structure operations, transforms, batch operations)
- Tools: UI constructs emitting primitives (Pointer, Blade, Pencil, Eraser, etc.)
- Selection system: type-aware selection items referencing Pulse IDs
- Snapping & grid: unified grid model across all editors
- Editor transactions: grouped undo/redo by user intent

#### Responsibilities / Components
- **Editor Core**: Shared model for selection, tools, snapping, focus, transactions
- **Editing Primitives**: Atomic operations (move, trim, split, add_note, quantise, etc.)
- **Tools**: Universal and editor-specific tools (Piano Roll: articulation tool; Audio: slice tool; Automation: curve tool)
- **Selection System**: Type-aware selection with Pulse ID references
- **Editor Types**: Piano Roll, Drum Sequencer, Audio Editor, Automation Editor, Clip Editor, ARA Editor, Video Editor

#### Inputs / Outputs / Dependencies
- **Aura → Pulse**: Editor creation, view state, selection, tools, transactions, edit primitives, gestures
- **Pulse → Aura**: Selection updates, view state updates, transaction confirmations, previews
- **Pulse ↔ Signal**: Preview waveforms, transient maps, warp previews, clip pipeline previews

#### Notes & Constraints
- All editors share unified conceptual model
- Editors exist conceptually in Aura; Pulse owns data
- Selection always references canonical Pulse IDs
- Tools do not manipulate data directly; generate primitives
- Transactions grouped by user intent (not raw primitives)
- Gesture integration: high-res streams decimated by Pulse

---

### `23-clip-launcher.md`

#### Purpose
Defines architecture for Clip Launcher: nonlinear performance and prototyping interface aligned with arrangement tracks, using Scenes and Slots.

#### Key Concepts
- Scene: column in Clip Launcher grouping Slots across Tracks
- Launcher Slot: optional Clip content at Track/Scene intersection
- Launcher state: active scene, active mode (arrangement/launcher/hybrid), per-track activation
- Scene Playthrough: sequential audition of scenes as full arrangement prototype
- Quantised launch: triggers scheduled for next quantisation boundary
- Launcher vs Arrangement: launcher overrides arrangement on per-Track basis

#### Responsibilities / Components
- **Scenes**: Columns grouping Slots, identified by sceneId
- **Launcher Slots**: Optional Clip references at Track/Scene intersections with launch parameters
- **Launcher State**: Global state (active scene, mode, per-track playback state, playthrough state)
- **Quantisation**: All triggers quantised to bar/beat/grid boundaries
- **Playback Modes**: Loop, play-once, play-then-return

#### Inputs / Outputs / Dependencies
- **Tracks**: Launcher rows align directly with Track order
- **Clips**: Slots reference Clips (arrangement clips or launcher-only clips)
- **Timebase**: Launcher uses same timebase as arrangement
- **Kits/Pad Workspaces**: Can populate Scenes automatically
- **Pulse → Signal**: Playback instructions for Slot Clips at quantised times

#### Notes & Constraints
- Launcher is overlay on existing Tracks, not parallel system
- Scenes do not overlap or occupy temporal positions
- Slots do not duplicate audio content (reference Clips only)
- Launcher overrides arrangement on per-Track basis when Slot playing
- Scene Playthrough plays scenes sequentially for audition without committing

---

### `24-rendering-and-offline-processing.md`

#### Purpose
Defines complete rendering architecture for realtime playback, anticipative prerendering, clip ghost renders, track freezing, bounce, stems, and full mixdown.

#### Key Concepts
- Render types: realtime, anticipative prerender, clip ghost renders, track freezing, bounce-in-place, stems, full mixdown
- Render Graph: sample-domain representation of Node Graph
- Timeline resolution: Pulse converts all musical-time to sample space
- Clip ghost rendering: materialisation of ClipEditPipelines
- Track freezing: renders to derived asset, replaces with FrozenNode
- Rendering and ARA: ARA must be resolved before building RenderGraph

#### Responsibilities / Components
- **Pulse**: Orchestrates rendering, constructs RenderGraph, resolves timeline, handles lineage and media creation
- **Signal**: Stateless execution engine receiving complete graph, automation, modulation, rendering blocks
- **Render Graph**: Sample-domain node representation with resolved parameters, media, automation
- **Clip Ghost Renders**: Materialised ClipEditPipeline results
- **Track Freezing**: Renders track to asset, replaces graph with FrozenNode

#### Inputs / Outputs / Dependencies
- **Pulse → Signal**: RenderGraph, sample-domain automation, modulation, ghost MIDI
- **Signal → Pulse**: Rendered buffers, progress, errors
- **Clip Pipelines**: Must integrate seamlessly with streaming
- **ARA**: Requires special handling (pre-analysis passes, region validation)
- **Cohorts**: Anticipative prerendering requires audio prefetching

#### Notes & Constraints
- Rendering deterministic and bit-stable
- Pulse constructs RenderGraph; Signal executes
- Signal never autonomously discovers media (Pulse provides explicit descriptors)
- ARA steps resolved before building RenderGraph
- Crash safety: Signal isolated; Pulse can restart and resume
- Ghost renders preserve lineage

---

### `25-control-surfaces-and-assistive-hardware.md`

#### Purpose
Defines architecture for control surfaces, MIDI controllers, and assistive hardware with universal capability model, dynamic mapping, Composer integration, and context awareness.

#### Key Concepts
- Hardware Device Model: uniform representation regardless of protocol (MIDI, HID, OSC, MCU/HUI)
- Device fingerprinting: Pulse sends fingerprints to Composer for device intelligence
- Mapping System: context-aware mapping rules (Arranger, Mixer, Launcher, Plugin UI, Clip Editor, Global)
- User overrides and learning: lightweight learn mode creating MappingRule overrides
- Hardware feedback: LEDs, displays, meters driven by Feedback Intents from Pulse
- Multi-device support: multiple devices simultaneously with separate profiles

#### Responsibilities / Components
- **Signal**: Device enumeration, low-level I/O, input parsing, output dispatch (transport layer only)
- **Pulse**: Device registry, mapping engine, mapping rule evaluation, feedback intent generation, context awareness
- **Composer**: Device intelligence (profiles, default mappings, protocol knowledge)
- **Aura**: Context signals, hardware management UI, learn mode UI

#### Inputs / Outputs / Dependencies
- **Signal → Pulse**: Device descriptors, raw control messages, hardware inputs
- **Pulse → Signal**: Feedback intents (LEDs, displays), mode-change messages
- **Pulse → Composer**: Device fingerprints
- **Composer → Pulse**: Device profiles, default mappings
- **Aura → Pulse**: Context signals (active view, focus state)

#### Notes & Constraints
- Signal never makes mapping decisions (only reports hardware and events)
- Pulse owns all mapping logic and context-aware behaviour
- Device profiles from Composer cached locally
- Hotplug/disconnect handled gracefully (preserves mappings)
- Context switching instant and non-intrusive
- Learn mode requires explicit user action

---

### `26-ux-and-visual-layer.md`

#### Purpose
Defines user experience and visual architecture for Loophole, describing workspaces, navigation, editing paradigms, and theming principles.

#### Key Concepts
- UX Philosophy: immediate musicality, depth without clutter, one mental model many views, context-aware, non-destructive, accessible
- Core Workspaces: Project Home, Arrangement, Clip Launcher, Editors, Mixer, Media Library, Diagnostics, Settings
- View Hierarchy: Global Header, Main Area (Primary/Secondary views), Global Footer
- Navigation: global hotkeys, command palette, context menus, breadcrumbs
- Arrangement View: Track headers, hybrid lanes model, clips & lanes, launcher overlay
- Editor Views: Piano Roll, Audio Editor, Automation Editor, Drum Sequencer

#### Responsibilities / Components
- **Workspaces**: Primary workspaces (Arrangement, Launcher, Mixer, Editors, Browser)
- **Navigation**: Hotkeys, command palette, context menus, breadcrumbs
- **Arrangement View**: Timeline with tracks, clips, lanes, launcher overlay
- **Editor Views**: Focused spaces for detailed manipulation
- **Media Browser**: Multiple browsing modes, search/filter, contextual recommendations
- **Theming**: Dark mode, high contrast, adjustable density, semantic colours

#### Inputs / Outputs / Dependencies
- **Aura**: Provides all UI and visualisation
- **Pulse**: Source of truth for all displayed data
- **Signal**: Provides metering/analysis data (via side channel)
- **Control Surfaces**: Context-aware mapping reflected in UI

#### Notes & Constraints
- UI must remain fluid during heavy DSP
- Never block when media offline/downloading
- Visual hints for handles, grids, selections
- Auto-scroll and auto-zoom where intuitive
- Layout persistence per screen set
- Accessible: scalable UI, keyboard-driven, screen-reader consideration

---

### `27-performance-and-live-recording.md`

#### Purpose
Defines architecture for live performance, clip launching, scene sequencing, and recording workflows with quantisation and live-safe behaviour.

#### Key Concepts
- Dual timeline model: Arrangement Timeline (linear) and Launcher Timeline (grid-based)
- Performance quantisation: all triggers quantised to bar/beat/grid (global, per-scene, or per-clip)
- Record modes: linear, loop (takes/layers), MIDI overdub, audio overdub, punch in/out
- Retrospective recording: always-on buffers for MIDI and optionally audio
- Follow actions: automatic navigation between scenes/clips
- Scene Playthrough: sequential audition of scenes

#### Responsibilities / Components
- **Clip Launching**: Quantised triggers with behaviour modes (start, stop, retrigger, legato)
- **Scene System**: Scenes launch Slots together, stop active clips
- **Follow Actions**: Automatic navigation (next scene/clip, random, repeat, constraints)
- **Record Modes**: Multiple modes (linear, loop takes/layers, overdub, punch)
- **Retrospective Capture**: Always-on buffers for "I played something but wasn't recording"
- **Live Quantisation**: Input timing correction, quantise-on-input optional

#### Inputs / Outputs / Dependencies
- **Aura → Pulse**: Clip/scene triggers, record mode changes, arm/disarm commands
- **Pulse → Signal**: Scheduled trigger instructions, record stream commands
- **Signal → Pulse**: Trigger execution confirmations, recording status, retrospective buffers
- **Performance Cohorts**: Live performance requires realtime cohort safety
- **Timebase**: All quantisation uses project timebase

#### Notes & Constraints
- Sample-accurate clip and scene triggering
- Pre-buffering and anticipative warm-up for safe switching
- Launcher state changes must never block
- Monitoring path must run in realtime cohort
- Retrospective buffers have maximum duration
- Follow actions must be deterministic and stable

---

### `28-windowing-and-plugin-ui.md`

#### Purpose
Describes architecture for managing plugin UI windows and Aura windows in multi-screen environments with position persistence and off-screen protection.

#### Key Concepts
- Screen Sets: set of Screen Descriptors for all attached displays, identified by ScreenSet ID
- Window Descriptors: logical window keys, geometry (absolute and normalised), modes (docked, floating, maximised, fullscreen, hidden)
- Layout Profiles: per-ScreenSet layouts keyed by (machineId, ScreenSetId)
- Off-screen protection: clamping rules ensuring windows always reachable
- Plugin UI windows: native plugin windows managed by Signal, positioned by Aura

#### Responsibilities / Components
- **Aura**: Queries OS for display configuration, constructs Screen Descriptors, manages layout profiles, applies clamping rules
- **Signal**: Creates and owns native plugin UI windows, attaches to OS parent
- **Pulse**: Coordinates plugin UI lifecycle, stores minimal UI state, carries position/size commands

#### Inputs / Outputs / Dependencies
- **Aura**: Decides window positions, manages layout profiles, ensures windows reachable
- **Signal**: Creates/manages plugin windows, respects size constraints
- **Pulse**: Coordinates between Aura and Signal, stores UI state
- **Plugin UI IPC**: Window creation, geometry changes, layout commands

#### Notes & Constraints
- Window positions recalled per screen configuration
- Automatic adjustment when screens added/removed
- Windows always reachable and resizable (clamping rules)
- Normalised geometry projects onto target display work area
- Fallback display chosen if target display missing
- Plugin windows constrained so all corners never off-screen

---

### `29-diagnostics-and-performance.md`

#### Purpose
Defines diagnostics, performance analysis, and engine introspection architecture providing visibility into all layers.

#### Key Concepts
- Four-layer diagnostics: Signal Metrics Engine, Pulse Aggregator, Composer Telemetry, Aura Diagnostic UX
- Plugin isolation and crash containment
- Per-node and graph metrics
- Anticipative engine diagnostics (progress, cohort membership, freeze candidates)
- Clip pipeline and ghost render diagnostics
- Hardware diagnostics (jitter, dropped packets, device health)

#### Responsibilities / Components
- **Signal Metrics Engine**: Captures raw performance metrics (per-node, graph, disk I/O, hardware, gesture streams, ARA)
- **Pulse Aggregator**: Converts metrics to structured diagnostic events, maintains rolling log
- **Plugin Isolation**: Crash containment, timeout detection, sandbox levels
- **Diagnostics UX**: CPU graphs, event log, node graph visualiser, hardware test mode
- **Composer Telemetry**: Anonymised telemetry for plugin stability, device jitter, performance patterns

#### Inputs / Outputs / Dependencies
- **Signal → Pulse**: Raw metrics, crash events, performance data
- **Pulse → Aura**: Diagnostic events, aggregated metrics
- **Pulse → Composer**: Anonymised telemetry (opt-in)
- **Composer → Pulse**: Improvement suggestions
- **Anticipative Engine**: Diagnostic hooks for progress and state

#### Notes & Constraints
- Diagnostics must not degrade real-time performance
- Plugin crashes must not crash Signal
- Signal crashes must not crash Pulse
- Crash logs and diagnostics stored for review
- User opt-in for telemetry to Composer
- Short-term history stored by default

---

### `30-scripting-and-extensibility.md`

#### Purpose
Defines architecture for scripting, automation, extensibility, and custom tooling with safe sandboxed execution.

#### Key Concepts
- JavaScript/TypeScript in sandboxed V8/QuickJS isolate running inside Pulse
- Sandbox architecture: no filesystem/network access, no blocking, timeouts, memory limits
- Permissions model: scripts declare required permissions
- Trigger model: event-driven (project, track/clip, timeline, parameter, hardware, diagnostics events)
- Custom UI: script panels in Aura
- Packaging: extensions as .loophole-extension.zip with manifest

#### Responsibilities / Components
- **Script Lifecycle**: Session scripts, global scripts, packaged extensions
- **Permissions Model**: Declared permissions enforced at API call boundary
- **Trigger System**: Event handlers for various domain events
- **Scripting API**: LoopholeAPI surface (project, tracks, clips, nodes, parameters, automation, mixer, launcher, hardware, events, ui, render, diagnostics)
- **Custom UI**: Script panels, modals, canvases in Aura
- **Composer Integration**: Extension discovery, ratings, update delivery

#### Inputs / Outputs / Dependencies
- **Scripts → Pulse**: API calls through LoopholeAPI host object
- **Pulse → Scripts**: Event notifications, API responses
- **Aura**: Provides UI surfaces for script panels
- **Composer**: Extension distribution, ratings, safety metadata
- Scripts never touch Signal or Aura directly

#### Notes & Constraints
- Scripts MUST never run on realtime thread
- Scripts cannot block rendering or allocate unbounded memory
- Pulse enforces max execution time, cooperative scheduling, API throttling
- Scripts cannot spawn threads or perform unsafe recursion
- All scripts sandboxed and isolated
- Scripts make commands to Pulse; Pulse updates model; Aura/Signal respond via IPC

---

### `31-video-architecture.md`

#### Purpose
Defines how Loophole supports video playback, editing, synchronisation, rendering, and analysis within the hybrid audio–media engine.

#### Key Concepts
- Video Assets: imported video files with metadata (container, codec, FPS, resolution, colour space)
- Video Lanes: lane type on Tracks (behave like audio lanes)
- Video Clips: time-bounded containers with in/out points, transforms, pipelines
- Video Node Graph: parallel to audio (decoder, transformer, compositor, colour nodes)
- Proxy system: lower resolution for editing
- Frame cache: Signal manages decoded frame cache
- Sample-accurate sync: video synchronised to Pulse timeline

#### Responsibilities / Components
- **Pulse**: Owns video metadata, proxy status, coordinates proxy requests
- **Signal**: Handles video decoding (worker threads), manages frame cache, provides frames to Aura
- **Video Nodes**: Processing chain for video (decoders, transformers, compositors)
- **Proxy System**: Generates lower-resolution proxies for editing
- **Video Editing**: Basic edits (trim, slip, stretch, cut, fade), transforms (position, scale, rotation, crop)

#### Inputs / Outputs / Dependencies
- **Pulse → Signal**: Frame requests, proxy generation requests, preview region updates
- **Signal → Pulse**: Decoded frame ready events, proxy progress, decode errors
- **Pulse ↔ Aura**: Preview frame handles, clip thumbnails, status updates
- **Video Automation**: Video parameters (opacity, scale, rotation) automated like audio parameters

#### Notes & Constraints
- Video decoding MUST never block realtime audio thread
- All decode work on worker threads
- Video playback framed around Pulse-managed timebase
- Proxy system required for large files
- Frame cache uses LRU eviction with memory budget
- Video editing initially minimal but structurally extensible
- ARA-style video analysis supported (not required for v1)

---

### `32-plugin-lifecycle-and-sandboxing.md`

#### Purpose
Defines complete plugin lifecycle, sandboxing model, and isolation semantics covering discovery, scanning, instance management, crash containment, and UI lifecycle.

#### Key Concepts
- Plugin identity and metadata (normalised across formats)
- Plugin scanning (initial deep scan, incremental scan, on-demand scan)
- Per-plugin-instance sandboxing (strict, moderate, disabled policies)
- Crash detection and recovery (silence output, auto-restart, user recovery)
- Deterministic vs non-deterministic classification (affects cohort assignment)
- Plugin UI lifecycle (window management, bounds persistence, multi-screen awareness)
- Format bridging (VST3, CLAP, AU with uniform wrapper)

#### Responsibilities / Components
- **Pulse**: Maintains plugin metadata, normalised identity, parameter mapping, coordinates with Composer
- **Signal**: Performs scanning (in sandbox process), creates/manages plugin instances, handles UI windows, monitors crashes
- **Composer**: Provides plugin categorisation, parameter aliases, deterministic classification, version mapping
- **Sandboxing**: Per-instance isolation (strict = separate process, moderate = shared process/isolated threads)
- **Crash Recovery**: Immediate response (silence), user-safe recovery, optional auto-restart

#### Inputs / Outputs / Dependencies
- **Pulse → Signal**: Plugin instance creation/destruction, parameter updates, UI creation commands
- **Signal → Pulse**: Plugin ready events, crash notifications, parameter changes, UI bounds changes
- **Pulse → Composer**: Plugin identity queries, deterministic classification requests
- **Composer → Pulse**: Plugin metadata, deterministic classification, parameter aliases
- **Aura**: Plugin UI positioning, layout management

#### Notes & Constraints
- Plugin crash must never crash Signal
- Plugin instantiation cannot block realtime thread
- Scanning occurs in dedicated sandbox process
- Deterministic plugins safe for anticipative rendering
- Plugin UI bounds stored per (machineId, ScreenSetId)
- Format bridging ensures consistent parameter automation behaviour
- State restoration: parameters first, then raw binary state

---

### `33-plugin-library-and-browser.md`

#### Purpose
Defines plugin library, browser, and organisational model focusing on user-facing plugin discovery, categorisation, search, and insertion workflows.

#### Key Concepts
- Plugin deduplication: variants (VST3/CLAP/AU) grouped under single PluginEntry
- User renaming: display names user-editable without affecting pluginId
- Plugin tags and categories: user and Composer-suggested organisation
- Search system: name, manufacturer, tag, parameter, feature, semantic search
- Plugin insertion: drag-drop, quick insert, intelligent insertion
- Plugin chains and templates: user-defined and Composer-suggested chains
- Handling duplicates and missing plugins: hiding duplicates, replacement suggestions

#### Responsibilities / Components
- **Plugin Library**: Canonical representation with plugins, categories, user overrides, Composer hints
- **Plugin Browser**: Multiple sections (Recent, Favourites, Categories, Tags, Manufacturers, All)
- **Search System**: Basic, tag, parameter, feature, and Composer-semantic search
- **Plugin Chains**: User-defined and factory chains of plugins
- **Presets**: Factory, user, project, and chain presets with cross-format migration

#### Inputs / Outputs / Dependencies
- **Pulse**: Maintains plugin library state, user overrides, categories, tags
- **Composer**: Provides categorisation, tags, parameter aliases, variant selection hints
- **Aura**: Provides browser UI, search interface, insertion workflows
- **Node Graph**: Plugins inserted as nodes via normal node creation

#### Notes & Constraints
- Plugin identity stability: renames do not break projects
- Hidden plugins still load in old projects
- Default variant selection based on format preference, sandbox support, feature completeness
- Search must be extremely fast
- Plugin cards show display name, tags, manufacturer, variant icon, favourite, preview
- Missing plugins: Pulse tries alternate variants, Composer provides suggestions

---

### `34-collaboration-architecture.md`

#### Purpose
Defines collaboration architecture for multiple users working on projects, supporting both asynchronous (version histories) and synchronous (real-time shared editing) models.

#### Key Concepts
- Asynchronous collaboration: version histories, branches, offline editing, merging revisions
- Synchronous collaboration: real-time shared editing with state synchronisation (Shared Pulse Server or Distributed Pulse with CRDT-like sync)
- Edit operations: serialised changes (operationId, timestamp, userId, domain, command, payload)
- Conflict resolution: last-writer-wins, domain-specific merging, explicit conflict UI
- Presence and awareness: user cursors, names, clip highlights, activity indicators
- Collaboration clock: logical timestamping ensuring causality ordering

#### Responsibilities / Components
- **Pulse**: Collaboration brain (authoritative state, edit operation sequencing, conflict detection/resolution, version metadata, presence management)
- **Aura**: Collaboration UIs (presence indicators, live cursors, merge conflict tools, user identity)
- **Edit Operations**: Atomic changes serialisable for merging and versioning
- **Versioning**: Project version IDs with parent relationships, branching, history graphs

#### Inputs / Outputs / Dependencies
- **Aura → Pulse**: Join/leave session, publish edits, request history/presence
- **Pulse → Aura**: Presence updates, remote edits, conflicts, resolution requirements
- **Pulse ↔ Pulse**: Collaboration flows (centralised or CRDT-inspired)
- **Signal**: Not network-aware; receives only resolved project commands

#### Notes & Constraints
- Collaboration builds on top of Pulse, not Signal
- Signal remains purely local engine
- Edit operations must be diffable, mergeable, timestamped
- Media files must be local or synchronised via external tools
- Collaboration messages encrypted over networks
- Opt-in and incremental (local workflows untouched)

---

### `35-composer-extended-intelligence.md`

#### Purpose
Expands Composer baseline to describe full future role including global knowledge aggregation, media/plugin intelligence, AI-assisted suggestions, and neural model integration.

#### Key Concepts
- Plugin intelligence: role classification, determinism classification, variant/version mapping, parameter aliasing, smart defaults
- Media intelligence: auto-tagging, similarity search using embeddings, cross-media relations, library-level recommendations
- Device intelligence: device database, templates, automatic mapping heuristics
- Project intelligence: structure/pattern analysis, mix insights, creative suggestions
- Semantic search: conceptual queries over plugins, samples, clips, presets, chains
- Privacy-first: opt-in, no raw audio uploads by default, anonymous, local vs global knowledge

#### Responsibilities / Components
- **Metadata Normalisation**: Clean, unify, standardise plugin/media/device/parameter metadata
- **Heuristic Inference**: Pattern recognition, classification, deterministic behaviour inference
- **ML/AI Models**: Neural models for embeddings, similarity, suggestions
- **Recommendation Engine**: Context-aware suggestions for plugins, samples, chains
- **Search & Semantic Embeddings**: Multi-dimensional vectors for similarity search

#### Inputs / Outputs / Dependencies
- **Pulse → Composer**: Plugin metadata requests, media analysis requests, search queries, device identification, project summaries
- **Composer → Pulse**: Metadata refinements, tag suggestions, replacement mappings, classification info, embeddings, suggestions
- All requests asynchronous and cancellable

#### Notes & Constraints
- Composer never blocks workflows (all requests asynchronous)
- All suggestions advisory and visible
- No raw audio uploads by default (only fingerprints, statistics, features)
- All sharing anonymous
- Versioned APIs with capability queries
- Local knowledge cache for offline operation

---

### `36-future-workflow-prototypes.md`

#### Purpose
Describes advanced, experimental workflow models that Loophole may support in future versions, ensuring core architecture remains compatible.

#### Key Concepts
- Five prototype classes: Gesture-Driven Editing, Spatial & Node-Space Workflows, Grid-Based Performance, AI-Augmented Creative Workflows, Multimodal Input & Interaction
- Experimental R&D: safe exploration without entangling core subsystems
- User-facing "Labs" features: experimental toggleable modules
- Architectural guarantees: must not break undo/redo, must not introduce non-deterministic project data, must be IPC-contained

#### Responsibilities / Components
- **Gesture-Driven Editing**: Elastic arranger, gesture-curve automation, instrument gestures
- **Spatial & Node-Space**: Freeform node canvas, spatial arrangement as composition tool
- **Grid-Based Performance**: Full performance grid view, live generative grid, multi-timeline grids
- **AI-Augmented Workflows**: Sketch to song flow, adaptive learning workspace, assistant overlay layer
- **Multimodal Input**: Voice commands, pen & tablet interaction, haptic feedback

#### Inputs / Outputs / Dependencies
- All prototypes must work through: Clips, Lanes, Nodes, Pulse model editing, IPC consistency
- Experimental systems sit outside core architecture but use stable IPC
- Signal exposes only deterministic, engine-safe hooks

#### Notes & Constraints
- Prototypes extend without fracturing UX
- Powered by core architecture
- Allow safe R&D experimentation
- Remain bound to undo/redo and project determinism
- Easy to enable/disable
- Not require engine changes for most prototypes

---

### `37-experimental-systems.md`

#### Purpose
Defines formal framework for designing, prototyping, testing, and evolving non-standard, high-risk, high-reward subsystems pushing DAW boundaries.

#### Key Concepts
- Experimental lifecycle: Research → Prototype → Beta → Stable or Retire
- Six categories: DSP Experiments, Generative & Creative Experiments, UI/Interaction Experiments, Structural Data Model Experiments, Composer/AI Experiments, External Integration Experiments
- Experimental sandbox: plugin-like containers, sidecar AI engines, Aura prototype workspace
- Constraints: must not break undo/redo, must not introduce non-deterministic project data, must be IPC-contained, must be user-visible and optional

#### Responsibilities / Components
- **DSP Experiments**: Novel processing techniques running offline or in sandbox (neural effects, experimental spectral processors)
- **Generative Experiments**: Systems that create rather than modify (generative MIDI, pattern forgers, AI improvisers)
- **UI/Interaction Experiments**: Experimental interaction models (VR/AR, pageless surfaces, force-directed canvases)
- **Structural Experiments**: Deeper model evolution (multi-dimensional timelines, non-linear clip networks)
- **Composer/AI Experiments**: Large-scale analysis, project fingerprinting, multi-agent assistants
- **External Integration**: Hybrid environments, scriptable automation, inter-app sync

#### Inputs / Outputs / Dependencies
- Experimental systems sit outside core architecture (philosophical boundary)
- Can exist as: UI modules in Aura, high-level modules in Pulse, external sidecar processes, offline render modules
- Signal should only expose deterministic, engine-safe hooks

#### Notes & Constraints
- Must never compromise: real-time safety, project determinism, IPC contract stability, user data integrity
- DSP experiments run offline or sandbox (never realtime until proven safe)
- Generative systems generate Clips/Lanes going through normal Pulse editing
- UI experiments live entirely in Aura, use stable IPC
- Structural experiments must wrap around existing Clips/Lanes/Tracks

---

### `38-ai-assisted-audio-tools.md`

#### Purpose
Defines architecture for AI-assisted tools that augment traditional workflows with suggestions, optional transforms, and non-blocking analysis.

#### Key Concepts
- AI as suggestive layer: visible suggestions, optional one-shot transforms, non-blocking analysis
- Interaction model: user invokes tool → Pulse requests → Composer responds → Aura displays → user chooses action → Pulse applies as normal edits
- Types of tools: Analysis & Insight, Editing Assistance, Performance/Musical Tools, Mixing & Processing Helpers, Library & Browser Intelligence
- Determinism constraints: offline renders deterministic per project state; AI outputs stored explicitly

#### Responsibilities / Components
- **Aura**: UI for invoking tools, visualisation of suggestions, editor overlays
- **Pulse**: Orchestrates AI requests, integrates suggestions into project model, maintains job queues
- **Signal**: Provides analysis input (stems, feature maps), lightweight analysis DSP
- **Composer**: Primary AI "brain" with models for tagging, pattern recognition, suggestion generation

#### Inputs / Outputs / Dependencies
- **Aura → Pulse**: AI requests (analysis, suggestions, transform previews)
- **Pulse → Composer**: Requests with scope, context, intent
- **Composer → Pulse**: Annotations, proposed edits, candidate transformations
- **Pulse → Aura**: Suggestion events
- All AI compute off audio thread, off UI thread for long tasks

#### Notes & Constraints
- AI features must never compromise audio safety, project determinism, or user control
- All AI operations expressed as new Clips/Lanes/Nodes or suggested edits
- AI tools opt-in (default behaviour fully manual)
- Heavy inference runs asynchronously and out-of-process
- Once AI suggestion accepted, stored as normal edits (deterministic)

---

### `39-sampler-and-virtual-instruments.md`

#### Purpose
Defines architecture for samplers and virtual instruments integrating as Nodes, supporting multi-sample instruments, MPE, and deep sampler capabilities.

#### Key Concepts
- Instrument Nodes: plugins or internal instruments (sampler, built-in synths) in NodeGraph
- MIDI & Lane integration: multiple MIDI Lanes per Track routing to InstrumentNodes
- Sampler architecture: SampleGroups containing Regions mapping samples to key/velocity ranges
- Drum kits: dedicated DrumKit mode with per-pad mapping
- Per-note expression: MPE, polyphonic aftertouch, per-note automation lanes
- Media Library integration: samples managed as first-class media

#### Responsibilities / Components
- **Instrument Nodes**: Instruments appear as nodes in NodeGraph (plugin instruments, sampler, internal instruments)
- **Sampler**: Core internal instrument with SampleGroups, Regions, multi-sample support
- **Drum Kits**: Specialised sampler mode for drum workflows
- **MIDI Routing**: Multiple Lanes can route to InstrumentNodes (single or multiple targets)
- **Per-Note Expression**: MPE and per-note expression throughout

#### Inputs / Outputs / Dependencies
- **MIDI Lanes → Instrument Nodes**: MIDI routing from Lanes to instruments
- **Instrument Nodes → Audio**: Audio output flows through Channel Nodes
- **Media Library**: Samples managed as media assets
- **Editors**: Piano Roll and Drum Sequencer query instrument capabilities
- **Composer**: May provide instrument categorisation and tagging

#### Notes & Constraints
- Instruments typically sit near start of Channel graph
- Sampler programmes stored as part of Node's persistent state
- Samples gracefully handle offline/remote media
- Drum instruments can be Sampler programmes or third-party plugins
- Per-note expression stored as per-note automation envelopes
- Instruments affect cohort assignment (deterministic vs non-deterministic, live interaction)

---

## 4. Cross-Document Relationships

### Explicit References Between Documents

The architecture documents contain numerous explicit cross-references:

- **01-overview.md** references `06-processing-cohorts-and-anticipative-rendering.md` for detailed cohort information.

- **02-pulse.md** references:
  - `10-parameters.md` for parameter aliasing
  - `11-node-graph.md` for node graph construction
  - `06-processing-cohorts-and-anticipative-rendering.md` for cohort assignment

- **03-signal.md** references:
  - `06-processing-cohorts-and-anticipative-rendering.md` for dual-engine architecture
  - Hardware I/O responsibilities link to `25-control-surfaces-and-assistive-hardware.md`

- **05-composer.md** explicitly references hardware integration sections that relate to `25-control-surfaces-and-assistive-hardware.md`.

- **06-processing-cohorts-and-anticipative-rendering.md** is referenced by multiple documents as the foundation for dual-engine processing.

- **08-tracks-lanes-and-roles.md** references:
  - `11-node-graph.md` for LaneStreamNode details

- **09-clips.md** references:
  - `08-tracks-lanes-and-roles.md` for Lane concepts

- **10-parameters.md** references:
  - `11-node-graph.md` for parameter ownership by nodes

- **11-node-graph.md** references:
  - `10-parameters.md` for parameter IDs
  - Mentions ARA-capable plugin nodes

- **12-mixer-and-channel-architecture.md** references:
  - `08-tracks-lanes-and-roles.md` for Track/Channel concepts
  - `11-node-graph.md` for Node types
  - `06-processing-cohorts-and-anticipative-rendering.md` for cohort assignment
  - Contains section 10 on Routing Architecture

- **13-media-architecture.md** explicitly relates to:
  - Tracks/Channels/Lanes documents
  - Clips document
  - Parameters and Automation document
  - Nodes document
  - Processing Cohorts document

- **14-channel-routing-and-send-return.md** references:
  - `08-tracks-lanes-and-roles.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - IPC specifications

- **15-audio-streaming-and-caching.md** references:
  - `13-media-architecture.md`
  - `09-clips.md`
  - `18-editing-and-nondestructive-layers.md` (as `16-editing-and-nondestructive-layers.md`)
  - `24-rendering-and-offline-processing.md` (as `20-rendering-and-offline-processing.md`)
  - `29-diagnostics-and-performance.md` (as `23-diagnostics-and-performance.md`)

- **16-timebase-tempo-and-groove.md** references:
  - `01-overview.md`
  - `07-project-versions-and-variants.md`

- **17-advanced-clips.md** references:
  - `09-clips.md` as foundational clip architecture
  - `16-timebase-tempo-and-groove.md` for timebase interaction

- **18-editing-and-nondestructive-layers.md** builds on:
  - `07-project-versions-and-variants.md`
  - `09-clips.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `13-media-architecture.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`

- **19-automation-and-modulation.md** builds on:
  - `10-parameters.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`
  - `18-editing-and-nondestructive-layers.md`

- **20-midi-architecture.md** builds on:
  - `09-clips.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`
  - `18-editing-and-nondestructive-layers.md`
  - `19-automation-and-modulation.md`

- **21-comping-architecture.md** builds on:
  - `08-tracks-lanes-and-roles.md`
  - `09-clips.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`
  - `18-editing-and-nondestructive-layers.md`
  - `19-automation-and-modulation.md`
  - `20-midi-architecture.md`
  - `11-node-graph.md` for LaneStreams

- **22-editor-architecture.md** complements:
  - `09-clips.md`
  - `17-advanced-clips.md` (as `15-advanced-clips.md`)
  - `18-editing-and-nondestructive-layers.md` (as `16-editing-and-nondestructive-layers.md`)
  - `19-automation-and-modulation.md` (as `17-automation-and-modulation.md`)
  - `20-midi-architecture.md` (as `18-midi-architecture.md`)
  - `21-comping-architecture.md` (as `19-comping-architecture.md`)
  - `24-rendering-and-offline-processing.md` (as `20-rendering-and-offline-processing.md`)
  - `25-control-surfaces-and-assistive-hardware.md` (as `21-control-surfaces-and-assistive-hardware.md`)
  - `26-ux-and-visual-layer.md`

- **23-clip-launcher.md** references:
  - Multiple track/clip/lane documents
  - Timebase documents
  - Media documents

- **24-rendering-and-offline-processing.md** interacts with:
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `11-node-graph.md`
  - `21-comping-architecture.md`
  - `19-automation-and-modulation.md`
  - `13-media-architecture.md`

- **25-control-surfaces-and-assistive-hardware.md** builds on:
  - `03-signal.md`
  - `04-aura.md`
  - `05-composer.md`
  - `10-parameters.md`
  - `11-node-graph.md`
  - `16-timebase-tempo-and-groove.md`
  - `19-automation-and-modulation.md`
  - `20-midi-architecture.md`

- **26-ux-and-visual-layer.md** builds on:
  - `04-aura.md`
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `08-tracks-lanes-and-roles.md`
  - `09-clips.md`
  - `10-parameters.md`
  - `13-media-architecture.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`
  - `18-editing-and-nondestructive-layers.md`
  - `19-automation-and-modulation.md`
  - `20-midi-architecture.md`
  - `21-comping-architecture.md`
  - `24-rendering-and-offline-processing.md`
  - `25-control-surfaces-and-assistive-hardware.md`
  - `28-windowing-and-plugin-ui.md`

- **27-performance-and-live-recording.md** complements:
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `14-channel-routing-and-send-return.md`
  - `22-editor-architecture.md`
  - `21-comping-architecture.md`
  - `12-mixer-and-channel-architecture.md`
  - `23-clip-launcher.md` (as `13-clip-launcher.md`)

- **28-windowing-and-plugin-ui.md** interacts with:
  - `04-aura.md`
  - `03-signal.md` (as `02-signal.md`)
  - `08-tracks-lanes-and-roles.md`
  - Plugin UI IPC specifications

- **29-diagnostics-and-performance.md** builds on:
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `18-editing-and-nondestructive-layers.md` (as `16-editing-and-nondestructive-layers.md`)
  - `24-rendering-and-offline-processing.md` (as `20-rendering-and-offline-processing.md`)
  - `25-control-surfaces-and-assistive-hardware.md` (as `21-control-surfaces-and-assistive-hardware.md`)

- **30-scripting-and-extensibility.md** builds on:
  - `03-signal.md`
  - `04-aura.md`
  - `05-composer.md`
  - `10-parameters.md`
  - `11-node-graph.md`
  - `16-timebase-tempo-and-groove.md`
  - `19-automation-and-modulation.md`
  - `20-midi-architecture.md`
  - `25-control-surfaces-and-assistive-hardware.md`

- **31-video-architecture.md** integrates with:
  - `08-tracks-lanes-and-roles.md`
  - `09-clips.md`
  - `10-parameters.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `16-timebase-tempo-and-groove.md`
  - `17-advanced-clips.md`
  - `18-editing-and-nondestructive-layers.md`
  - `24-rendering-and-offline-processing.md`
  - `25-control-surfaces-and-assistive-hardware.md`
  - `29-diagnostics-and-performance.md`

- **32-plugin-lifecycle-and-sandboxing.md** complements:
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `14-channel-routing-and-send-return.md`
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `29-diagnostics-and-performance.md`
  - `25-control-surfaces-and-assistive-hardware.md`
  - `05-composer.md`
  - `33-plugin-library-and-browser.md`

- **33-plugin-library-and-browser.md** provides user-facing counterpart to:
  - `32-plugin-lifecycle-and-sandboxing.md`
  - Integrates with `05-composer.md`
  - `13-media-architecture.md`
  - `11-node-graph.md`
  - `26-ux-and-visual-layer.md`

- **34-collaboration-architecture.md** complements:
  - `02-pulse.md`
  - `03-signal.md`
  - `04-aura.md`
  - Multiple track/clip/parameter documents
  - `24-rendering-and-offline-processing.md`
  - `26-ux-and-visual-layer.md`

- **35-composer-extended-intelligence.md** expands:
  - `05-composer.md`
  - `33-plugin-library-and-browser.md`
  - `13-media-architecture.md`
  - `38-ai-assisted-audio-tools.md`
  - `06-processing-cohorts-and-anticipative-rendering.md`

- **36-future-workflow-prototypes.md** complements:
  - `26-ux-and-visual-layer.md`
  - `22-editor-architecture.md`
  - `13-media-architecture.md`
  - `33-plugin-library-and-browser.md`
  - `38-ai-assisted-audio-tools.md`
  - `34-collaboration-architecture.md`

- **37-experimental-systems.md** complements:
  - `36-future-workflow-prototypes.md`
  - `38-ai-assisted-audio-tools.md`
  - `29-diagnostics-and-performance.md`
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `05-composer.md` and `35-composer-extended-intelligence.md`
  - Core Signal, Pulse, Aura documents

- **38-ai-assisted-audio-tools.md** complements:
  - `06-processing-cohorts-and-anticipative-rendering.md`
  - `11-node-graph.md`
  - `20-midi-architecture.md`
  - `21-comping-architecture.md`
  - `22-editor-architecture.md`
  - `29-diagnostics-and-performance.md`
  - `33-plugin-library-and-browser.md`
  - `05-composer.md`

- **39-sampler-and-virtual-instruments.md** complements:
  - `08-tracks-lanes-and-roles.md`
  - `09-clips.md`
  - `10-parameters.md`
  - `11-node-graph.md`
  - `12-mixer-and-channel-architecture.md`
  - `20-midi-architecture.md`
  - `22-editor-architecture.md`
  - `13-media-architecture.md`
  - `33-plugin-library-and-browser.md`

### Forward Dependencies

The documents follow a clear dependency hierarchy:

**Foundational Layer (01-07):**
- All other documents depend on `01-overview.md` for core architectural principles
- `02-pulse.md`, `03-signal.md`, `04-aura.md`, `05-composer.md` define core component responsibilities
- `06-processing-cohorts-and-anticipative-rendering.md` is foundational for performance architecture
- `07-project-versions-and-variants.md` provides project model foundation

**Structural Layer (08-15):**
- Depends on foundational documents
- `08-tracks-lanes-and-roles.md` provides Track/Channel/Lane model used throughout
- `09-clips.md` depends on `08-tracks-lanes-and-roles.md`
- `11-node-graph.md` provides Node model referenced by many documents
- `12-mixer-and-channel-architecture.md` extends Node concepts

**Timeline Layer (16-22):**
- Depends on structural layer
- `16-timebase-tempo-and-groove.md` provides time model used by all timeline features
- `17-advanced-clips.md` extends `09-clips.md`
- `18-editing-and-nondestructive-layers.md` builds on multiple structural documents
- `19-automation-and-modulation.md` depends on parameters and node models
- `20-midi-architecture.md` integrates with clips, timebase, and editing
- `21-comping-architecture.md` builds on tracks, clips, and editing models
- `22-editor-architecture.md` provides unified editing platform

**Performance & Integration Layer (23-39):**
- Depends on timeline and structural layers
- `23-clip-launcher.md` builds on tracks and clips
- `24-rendering-and-offline-processing.md` integrates with cohorts, nodes, comping
- `25-control-surfaces-and-assistive-hardware.md` integrates with signal, aura, composer
- Later documents (34-39) build on earlier ones and reference future capabilities

### Shared Structures

Several architectural patterns are shared across documents:

1. **IPC Contract Pattern**: Documents consistently define IPC responsibilities between Pulse, Signal, and Aura using command/event patterns.

2. **Pulse as Authority**: Nearly all documents reinforce Pulse as the authoritative source of truth for project state.

3. **Determinism Requirements**: Multiple documents emphasise deterministic behaviour, especially for rendering and project state.

4. **Non-Destructive Editing**: Documents consistently use pipeline and layer models for non-destructive operations.

5. **Cohort Awareness**: Documents that involve processing consistently reference the processing cohorts architecture.

6. **Composer Integration**: Documents consistently treat Composer as optional, advisory, and accessed via Pulse.

---

## 5. Identified TODOs or Underspecified Points

### Explicit TODOs

- **00-index.md**: Notes that items marked "(planned)" are intentionally left for upcoming documentation passes; items marked "(future)" are long-range and do not constrain v1.

- **02-pulse.md**: Section 13 "Future Extensions" lists possible future additions (modulation system, graph-based Clip-dependent routing, multi-engine bridging, etc.) but these are not TODOs, rather forward-looking possibilities.

- **03-signal.md**: Section 13 "Future Extensions" lists potential evolutions (multi-engine distributed processing, GPU-accelerated DSP nodes, etc.) as future possibilities, not immediate TODOs.

- **05-composer.md**: Section 12 "Fut
