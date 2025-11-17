# Architecture Inbox

This document collects ideas, observations, discussions, and speculative features
that arise during Loophole’s design process. Items recorded here are not yet
commitments. They exist to preserve insight without blocking ongoing work or
disrupting architectural flow.

Entries in this inbox may be refined, merged, escalated into formal
architecture documents, incorporated into IPC specifications, or discarded.

---

## Format for New Entries

Each new idea should follow this structure:

```
## <Short Title>
**Tag:** <Engine / Pulse / Aura / IPC / Workflow / DSP / UX / Composer / Misc>  
**Priority:** P1 | P2 | P3  
**Status:** proposed | accepted | incorporated | rejected  
A concise description of the idea, why it matters, and when it should be
considered. Include cross-references if relevant.
```

---

## Entries

### Node Determinism Classification
**Tag:** Composer / Signal / Pulse  
**Priority:** P1  
**Status:** incorporated  
Composer should collect behavioural telemetry to determine whether a plugin
is deterministic or non-deterministic. This allows Pulse to safely place nodes
into the anticipative cohort and Signal to avoid unpredictable behaviour in the
background engine.

### Dynamic Cohort Switching UX Indicators
**Tag:** Aura / UX  
**Priority:** P3  
**Status:** proposed  
Aura may display a small indicator showing whether a track or node is running
in the live or anticipative domain. Useful for debugging performance and improving
user intuition around engine behaviour.

### High-Rate Gesture Stream
**Tag:** IPC / Pulse / Signal  
**Priority:** P2  
**Status:** proposed  
Add a dedicated IPC side-channel for ultra-fast parameter gestures (e.g., fast knob
movements), to avoid JSON overhead during continuous control. Pulse should still
validate and route gestures but not block real-time flow.

### Node Identity Aliasing
**Tag:** Pulse / Composer  
**Priority:** P2  
**Status:** proposed  
Support alias definitions so projects remain stable if a plugin changes identity
after an update. Composer may help suggest identity mappings across versions
and formats.

### Track-Level Freeze Visualisation
**Tag:** Aura / UX / Signal  
**Priority:** P3  
**Status:** proposed  
Visually represent when anticipative rendering is effectively “freezing” a track so
users understand the balance between live and pre-rendered DSP.

### Multi-Clip Lane Expansion
**Tag:** Pulse / Aura  
**Priority:** P3  
**Status:** proposed  
Allow clips to define new Lanes that are not part of the Track's static layout,
enabling per-clip experimentation with audio or MIDI layering.

### SQLite-Based Storage Architecture
**Tag:** Composer / Misc  
**Priority:** P2  
**Status:** proposed  
Use SQLite (or similar SQL database) for storage of project data and settings/config that don't require optimised data streams (e.g., excluding raw audio samples). Requires a general database for Composer sync and settings, plus investigation into whether project files should be or contain SQLite databases. Need to evaluate constraints and limitations of this approach.

### Gesture-Based Tool Selection
**Tag:** Aura / UX  
**Priority:** P3  
**Status:** proposed  
Enable gesture-based tool selection in editors (e.g., piano roll) by holding an action key and performing mouse gestures to switch modes. Maintain keyboard shortcut fallbacks (e.g., number keys) for traditional workflows. Improves workflow fluidity for users who prefer gesture-based interaction.

### Context-Aware Plugin Suggestions
**Tag:** Composer / Aura / UX  
**Priority:** P2  
**Status:** proposed  
Implement processing contexts that detect track characteristics (e.g., drums, vocals, bass) and prioritize relevant plugin categories when loading FX. For example, a drum track would prioritize Drum category plugins, followed by Compressors, Shapers, etc. Reduces cognitive load and speeds up workflow by surfacing contextually relevant nodes first.

### Visual Plugin Browser
**Tag:** Composer / Aura / UX  
**Priority:** P2  
**Status:** proposed  
Transform plugin/node browsing into a visual experience with interface thumbnail captures, intelligently-picked conceptual colors (matching the plugin's "feel" rather than static collection colors), thematic icons from a wide selection (potentially auto-applied), routing diagrams (e.g., multi-out configurations), determinism indicators, and plugin stability metrics. Enables faster visual recognition and selection of nodes.

### Fine-Grained Track Nesting Controls
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Design extensive UI elements for fine-grained control of track nesting and folder organization. Existing DAW implementations make it awkward to move tracks between folders, into nested structures, or adjust nesting levels. Aura should provide intuitive drag-and-drop, keyboard shortcuts, and visual feedback for reorganizing track hierarchies without disrupting workflow.

### Extensible Action and Shortcut Configuration
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Implement fully extensible and customizable keyboard, action, and macro shortcut configuration system. Support mapping any control signature (mouse, keyboard, MIDI controller, etc.) to any action with simple conflict resolution. Allow multiple gestures to map to a single action, with gestures contextual to the active window. Enable in-place action discovery with optional hover tooltips and easy in-place assignment of shortcuts. Empowers users to create personalized workflows and adapt the interface to their preferred input methods.

### Predictable UI Layout and Window Management
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Design Aura's UI layout to be visually similar to other DAWs but with a much more predictable interaction model. Main arrangement windows contain the full sequence, with an optional clip launcher interface able to replace or partially cover it (similar to Studio One 7). Clip/lane editors, mixer, media library, etc. should have easy shortcuts by default and replace the whole internal window space for focused editing work. A secondary level of editor window can be attached in-place where appropriate for smaller editing facilities, similar to Bitwig/Ableton's bottom strip. This layered approach provides both full-screen focus and contextual secondary tools without disrupting the primary workflow.

### Multiple Mixer and Console Views
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Enable users to build different collections of Channels across multiple mixer/console views. For example, instruments and audio tracks in one view, group busses in another, send effects in another. This allows users to organize their mixing workflow by function or signal flow, reducing visual clutter and improving focus on specific mixing tasks. Rules for categorizing tracks into collections can be defined and refined later, but the core capability should support flexible channel grouping and filtering. Separate these views into easily accessible tabs or docked windows.

### Two-Layer UI Architecture with Theme Separation
**Tag:** Aura / UX  
**Priority:** P2  
**Status:** proposed  
Build Aura's UI layer with two distinct layers of markup and CSS. The lowest layer defines the basic visual structure of all UI elements without styling intent, establishing only layout, positioning, and structural relationships. A theme layer sits on top and paints over the basic structure, applying colors, fonts, borders, shadows, and all visual styling. This separation allows complete retheming of the entire UI without having to redefine core basic layout, enabling users and third parties to create custom themes while maintaining structural consistency and reducing maintenance burden.

### UI Modding and Extension Ecosystem
**Tag:** Aura / UX / Pulse / Workflow  
**Priority:** P2  
**Status:** proposed  
Give users the ability to "mod" the UI by building new creative interfaces to the standard set of features provided by Pulse. This should work similarly to the VSCode plugin ecosystem, with a well-defined extension API and marketplace. These extensions won't be able to extend the project data model, maintaining data integrity and compatibility. However, we can provide often-requested Pulse extensions later down the line to accommodate more complex UI ideas that require additional backend support. Enables community-driven innovation and customization while preserving core system stability.

### Multiple MIDI Editor View Types
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Design MIDI clip/lane editor views with support for multiple interface types: standard piano roll, drum sequencer, and room for more creative MIDI interpretations. All variants must be able to control notes and automation, ensuring consistent data access regardless of the visual representation. Beyond these core requirements, additional creative interfaces become an exercise in Aura extension building, leveraging the UI modding ecosystem. Discussion to be scheduled for brainstorming other possible interfaces for editing MIDI data, exploring alternative visualizations and interaction paradigms that could enhance workflow or inspire new creative approaches.

### MIDI Humanization Controls
**Tag:** Pulse / Aura / Signal / DSP  
**Priority:** P2  
**Status:** proposed  
Implement global and track-local humanization controls for MIDI playback. Apply a pre-buffered randomization pass prior to playback that affects timing and velocity without modifying the note data stored in the project. This makes quantized notes feel more natural and human-like during playback while preserving the original quantized data for editing. Experiment with intelligent placement of notes and velocity based on analysis of real recordings to create more authentic-sounding humanization patterns. The humanization should be applied in the playback pipeline, ensuring it can be toggled on/off and adjusted without affecting project data integrity. Can be represented visually with a background blur effect around notes to indicate the possible range of humanized note timings / velocity.

### Pre-Programmable Quantization Rulesets
**Tag:** Aura / UX / Workflow / Pulse  
**Priority:** P2  
**Status:** proposed  
Enable users to create and save pre-programmable quantization rulesets as quantize patches that can be easily selected. Support various quantization styles including "soft" quantizing (applying to only a small percentage), swing, groove templates, and other timing variations. The quantize UI should be extremely minimal by default, with advanced options only made available in a contextual modal when needed. This keeps the interface clean and fast for common operations while providing full flexibility for power users. Quantize patches can be shared, saved, and quickly applied to selections, improving workflow efficiency and enabling consistent quantization styles across projects.

### Channel Headroom Knobs and Gain Staging Analysis
**Tag:** Pulse / Aura / Signal / DSP / Workflow  
**Priority:** P2  
**Status:** proposed  
Concept to explore: implement headroom knobs on each channel allowing seamless pre-gain and equal post negative gain on either side of the effects processing list. This enables precise gain staging by adjusting input levels before processing and compensating output levels after processing, maintaining optimal signal levels throughout the chain. Complement this with detailed analysis of levels and loudness per-channel, providing real-time visual feedback on peak levels, RMS, LUFS, and other metering data. This combination allows for much simpler gain staging at mix time, giving users clear visibility and control over signal levels at every stage of processing without needing to insert dedicated gain plugins.

### Plugin Gain Comparison Indicators
**Tag:** Aura / UX / Pulse / Signal / Workflow  
**Priority:** P2  
**Status:** proposed  
Offer users a visual guide on effects plugins showing how much gain each plugin is adding or subtracting, displayed directly within the node graph view. This eliminates the need for external tools or manual level checking, providing immediate feedback on gain changes at each processing stage. The indicator should show both input and output levels with a clear visual comparison (e.g., before/after meters, gain delta display, or color-coded indicators). Potentially allow users to apply an automatic gain adjustment to compensate for plugin gain changes, maintaining consistent levels throughout the processing graph and simplifying gain staging workflows.

### Modulation System
**Tag:** Pulse / Aura / Signal / DSP / Workflow  
**Priority:** P2  
**Status:** proposed  
Implement a modulation system similar to Bitwig's approach, with modulation generators (LFOs, envelopes, step sequencers, etc.) that can be routed to any parameter on any node, instrument, or console channel. This provides powerful real-time control and automation capabilities, enabling complex parameter modulation without requiring traditional automation lanes. The system should support multiple modulation sources per parameter, with visual feedback showing modulation routing and active modulations. This enables creative sound design workflows and dynamic parameter control that goes beyond static automation curves.

### Built-In Professional Analysis Tools
**Tag:** Pulse / Aura / Signal / DSP / Workflow  
**Priority:** P2  
**Status:** proposed  
Provide a full suite of professional-grade analysis tools built directly into the system, including spectrum analyzers, waveform displays, FFT analysis, loudness metering (LUFS, LU, etc.), phase correlation, stereo imaging, and other advanced analysis capabilities. Since these are built-in rather than external plugins, we can optimize data flow internally, sharing analysis data efficiently across the system without redundant processing. This eliminates the need for users to rely on external analysis plugins, provides consistent analysis capabilities across all projects, and enables tighter integration with the mixing and mastering workflow.

### Customizable View Layouts and User Modes
**Tag:** Aura / UX / Workflow  
**Priority:** P2  
**Status:** proposed  
Enable customizable view layouts with the ability to enable or disable features within the main view, allowing users to create and save custom "modes" optimized for different use cases. Users should be able to build modes for writing, recording, mixing, mastering, and other workflows, showing or hiding specific UI elements, panels, and tools as needed. These modes can be saved, shared, and quickly switched between, reducing visual clutter and focusing the interface on the task at hand. This empowers users to create personalized workspace configurations that match their workflow preferences and the specific demands of each production phase.

### Bounced Track Source Encapsulation and Reopening
**Tag:** Pulse / Aura / Workflow  
**Priority:** P2  
**Status:** proposed  
Implement a system to encapsulate and hide the original source of "bounced" tracks, extending beyond simple bounce-in-place to include cases where instruments generate source material that is subsequently cut up and processed. The system should maintain institutional support for "reopening" the original source generator even long after tracks have been bounced, enabling users to return to and modify the original source material. This requires tracking source relationships, preserving generator state and parameters, and providing UI mechanisms to access and restore original sources. Enables non-destructive workflows where users can iterate on source material without losing the ability to return to earlier stages of production.

