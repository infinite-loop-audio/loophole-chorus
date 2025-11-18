# Architecture Inbox  
Loophole — Full Feature & Architecture Backlog  
*(Structured by Domain → Functional Areas, with priority tags)*

This inbox contains **all planned, desired and future features** for Loophole DAW.  
Its purpose is to ensure **complete architectural coverage**, so that nothing in the final product is blocked by missing early design decisions.

Priority markers:  
- **[CORE]** – Needed to finalise architecture & support v1  
- **[NEXT]** – High-value for v1.1–v2; should influence architecture now  
- **[ADVANCED]** – Future versions; architecture must not block  
- **[OPTIONAL]** – Experimental; nice-to-have or speculative

---

# 01 — Project & Session

## 1.1 Project Model  
- [CORE] Project templates (track templates, starter kits)  
- [NEXT] Per-track project versions / Track Versions  
- [NEXT] Alternate project arrangements (A/B arrangements)  
- [NEXT] Project “variants” or branches (nonlinear project evolution)  
- [ADVANCED] Collaboration-safe project model (track locking, merges, authorship metadata)  
- [OPTIONAL] Cloud project workspaces (real-time collaboration, conflict resolution)

## 1.2 Session & Global State  
- [CORE] Detailed background task system (media downloads, analysis jobs, rendering, recovery)  
- [CORE] More granular session warnings & indicators (dropouts, missing media, undo corruption warnings)  
- [NEXT] Session profiles (performance mode / editing mode presets)  
- [NEXT] Power-saving or low-load mode for laptops  
- [ADVANCED] Session “sandboxing” for trying destructive edits safely  
- [OPTIONAL] Multi-session tabbed environment (like Reaper)

---

# 02 — Timebase, Tempo & Groove

## 2.1 Tempo & Signature  
- [CORE] Tempo detection from audio  
- [NEXT] Per-Clip tempo maps  
- [NEXT] Tempo import/export from external sources (Ableton warp markers, Beatgrid files)  
- [ADVANCED] Tempo-follow (Ableton-style “Ableton Link++”, dynamic tempo following performance)  

## 2.2 Groove & Swing  
- [CORE] Groove templates (global + per-Clip + per-Lane)  
- [CORE] Groove extraction from audio & MIDI  
- [NEXT] Groove library (user-created groove presets)  
- [NEXT] Smart groove editor (adjust swing per subdivision)  
- [ADVANCED] Machine-learned groove inference (Composer)

---

# 03 — Tracks

## 3.1 Track Roles & Metadata  
- [CORE] Track folders with role metadata (drum group, vocal bus, etc.)  
- [NEXT] Track role–aware suggestion engine (Composer-driven)  
- [ADVANCED] Context-aware auto-track creation (e.g. detect a vocal file, create a vocal channel strip)

## 3.2 Track Variants  
- [NEXT] A/B/C variations of Track settings (FX, chain, automation)  
- [ADVANCED] Full “Track Versions” system (Logic/Cubase style)

---

# 04 — Lanes

## 4.1 Lane Types  
- [CORE] Explicit note for: lane capabilities (audio/MIDI/video/etc.)  
- [NEXT] Multi-output instrument lanes (per-output channelisation)  
- [ADVANCED] Hybrid lanes with layered materials (e.g. additive audio+MIDI lane)

## 4.2 Lane Editing Tools  
- [NEXT] Multi-lane “ghosting” view (reference lanes while editing one)  
- [ADVANCED] Lane grouping & linked editing tools  

---

# 05 — Clips

## 5.1 Audio Clip Editing  
- [CORE] Crossfades (automatic + manual)  
- [CORE] Clip gain envelopes  
- [CORE] Slip editing inside a fixed boundary  
- [CORE] Time-stretch modes (elastique quality, real-time, offline)  
- [NEXT] Warp markers with tempo-locked audio  
- [NEXT] Formant-preserving pitch-shift  
- [ADVANCED] Per-Clip spectral edits  
- [ADVANCED] Multi-source Clip layering inside one Clip container  

## 5.2 MIDI Clip Editing  
- [CORE] Scale snapping & mode support  
- [CORE] Basic quantise operations  
- [CORE] Velocity editing tools (bars, curves)  
- [NEXT] Chord guides / chord templates lane  
- [NEXT] Groove application to MIDI  
- [NEXT] Per-note expression (MPE)  
- [ADVANCED] MIDI transform tools (mirror, invert, density scaling, etc.)  
- [ADVANCED] MIDI pattern recognition (Composer could suggest MIDI based on project context)

## 5.3 Clip Containers  
- [NEXT] Clip “containers” (Bitwig / Ableton-style nested clip containers)  
- [ADVANCED] Modulation inside Clips (like Bitwig Clip modulators)

---

# 06 — Media & Library

## 6.1 Storage & Availability  
- [CORE] External drive offline/online transitions  
- [CORE] Cloud-backed storage handling (Dropbox/iCloud hydration)  
- [CORE] Placeholder media for missing files  
- [NEXT] Media “consolidate project” workflow  
- [NEXT] Duplicate detection & cleanup tools  
- [ADVANCED] Cross-drive media federation (global index abstraction)

## 6.2 Browsing Modes  
- [CORE] Visual cluster view (XO-style point cloud)  
- [CORE] Waveform thumbnail grid  
- [CORE] Folder/pack browser  
- [NEXT] Hypergrid drum/texture browser  
- [NEXT] Galaxy/spectrogram visualisation  
- [ADVANCED] Gesture-driven browsing (scrub curves → find matching envelopes)  
- [OPTIONAL] 3D immersive browser (VR/AR exploration)

## 6.3 Search, Tags & Metadata  
- [CORE] Semantic search (“warm pad”, “aggressive kick”)  
- [CORE] BPM/key compatibility filters  
- [NEXT] Smart Collections (metadata-driven bins)  
- [NEXT] Tagging workflows (user tagging + Composer tagging)  
- [ADVANCED] Auto-tagging via ML (Composer)  
- [OPTIONAL] Community tagging (cloud)

## 6.4 Kit Builder & Workspaces  
- [CORE] Kits as first-class objects  
- [CORE] Pad Workspace integration  
- [NEXT] Drum/vocal/texture kit autogeneration  
- [ADVANCED] Section-aware kit builder  
- [ADVANCED] Multi-kit composition systems

---

# 07 — Nodes (Processing Graph)

## 7.1 Node Types & Expansions  
- [CORE] SamplerNode with slicing support  
- [CORE] Send/Return nodes (already architected)  
- [NEXT] PitchNode with formant shifter  
- [NEXT] GranularNode (creative processor)  
- [ADVANCED] SpectralNode (FFT-based processing)  
- [ADVANCED] AI-driven adaptive Node (Composer integration)

## 7.2 Node Graph Editing  
- [CORE] Drag-and-drop Node reordering  
- [NEXT] Node “containers” (subchains, macros)  
- [ADVANCED] Node graph presets  
- [ADVANCED] Node graph “morphing” (scene interpolation)

---

# 08 — Channels & Mixer

## 8.1 Mixer Features  
- [CORE] Clip-safe gain staging tools  
- [NEXT] Mixer snapshots / A-B-C comparison  
- [NEXT] Mid/Side metering  
- [ADVANCED] Mixer morphing (interpolating between states)  
- [ADVANCED] Integrated reference track tools

## 8.2 Routing  
- [CORE] Arbitrary send-in-chain routing model (already specced)  
- [NEXT] Track-to-track audio routing matrix  
- [ADVANCED] Surround / spatial audio architecture  
- [OPTIONAL] Object-based mixing (Atmos-style)

---

# 09 — Automation

## 9.1 Curves & Editing  
- [CORE] Bezier & tension curves (already added)  
- [CORE] Range-level curve operations  
- [NEXT] Multi-parameter automation groups  
- [NEXT] Automation transforms (scale, offset, randomise)  
- [ADVANCED] Automation “clips” / envelopes inside Clips  
- [ADVANCED] Automation morphing

---

# 10 — MIDI Domain

## 10.1 MIDI Recording & Capture  
- [CORE] Retrospective MIDI recording  
- [CORE] Per-track input filtering (channels, MPE, CC)  
- [NEXT] Multi-input merging (multiple controllers into one lane)  
- [NEXT] MIDI articulation system (CC switching, keyswitch mapping)  
- [ADVANCED] MIDI capture “shadow buffer” like Ableton Capture

## 10.2 MIDI Editing Extensions  
- [CORE] Basic piano roll  
- [NEXT] MPE per-note bends & slides  
- [NEXT] Per-note CCs  
- [ADVANCED] MIDI scripting/modulators  
- [ADVANCED] MIDI generative tools (Composer-driven)

---

# 11 — Comping

## 11.1 Audio Comping  
- [CORE] Take lanes  
- [CORE] Swipe comping  
- [NEXT] Linked multi-track comping  
- [NEXT] Alternates/A/B comp passes  
- [ADVANCED] Comp suggestion system (Composer learns user preferences)

## 11.2 MIDI Comping  
- [NEXT] Segmented MIDI comping  
- [ADVANCED] Merge-tools for polyphonic takes

---

# 12 — Clip Launcher & Scenes

*(Most architecture already defined; this is for additional items)*

## 12.1 Launcher Extensions  
- [CORE] Scene Playthrough (implemented in arch doc)  
- [NEXT] Per-Scene FX states  
- [NEXT] Scene variations (A/B/C)  
- [ADVANCED] Scene-to-Section conversion tools  
- [ADVANCED] Arrangement Blocks (logic-like arrangement boxes) that map back to Scenes

## 12.2 Performance Features  
- [NEXT] Follow Actions (Ableton-style random/next/previous)  
- [NEXT] Stochastic slot launching  
- [ADVANCED] Performance “Macros” (state recall during performance)

---

# 13 — Rendering, Freeze & Offline Processing

## 13.1 Rendering  
- [CORE] Freeze render (already specified)  
- [NEXT] Multi-stage freeze (instrument only, full FX, tail preserve)  
- [NEXT] Per-Lane or per-Node rendering  
- [NEXT] Stems export matrix  
- [ADVANCED] Parallel offline render for heavy jobs  
- [OPTIONAL] GPU-accelerated render pipeline

## 13.2 Offline Processing  
- [NEXT] Render-in-place per Clip  
- [NEXT] Apply FX destructively with undo-safe snapshots  
- [ADVANCED] Per-Clip FFT processing tools

---

# 14 — History / Undo

## 14.1 Undo Features  
- [CORE] Global + contextual undo stacks  
- [NEXT] Mergeable operations (macro undo)  
- [ADVANCED] Timeline “branching undo” (Reaper-like)  

---

# 15 — Control Surfaces & Mapping

## 15.1 Mapping  
- [CORE] Learn mode for parameters  
- [NEXT] Hardware surface templates  
- [NEXT] Bi-directional feedback (lights, motorised faders)  
- [ADVANCED] OSC communication  
- [ADVANCED] Advanced scriptable surface API  
- [OPTIONAL] EUCON / HUI support

---

# 16 — Video

## 16.1 Video Editing & Sync  
- [CORE] Video track with offset  
- [NEXT] Frame-accurate snapping  
- [NEXT] Subtitle/cue lanes  
- [ADVANCED] ADR workflows  
- [ADVANCED] Video reconform tools  

---

# 17 — Performance Diagnostics

## 17.1 Engine Telemetry  
- [CORE] Node-level CPU reports  
- [CORE] Cohort heatmap (anticipative load map)  
- [NEXT] Per-Track performance analysis  
- [ADVANCED] ML-based performance prediction (Composer helps)

---

# 18 — Scripting & Extensibility

## 18.1 Script Interfaces  
- [NEXT] Action macros  
- [ADVANCED] Embedded scripting (JS/Lua/Python)  
- [ADVANCED] Event listeners for Pulse model mutations  
- [OPTIONAL] Procedural composition tools  

---

# 19 — Collaboration (Future)

## 19.1 Async Collaboration  
- [ADVANCED] Cloud project sync  
- [ADVANCED] Conflict resolution engine  
- [OPTIONAL] Live collaborative editing (Google Docs-style DAW)

---

# 20 — UX & Visual Layer

## 20.1 Browser / Mixer / Editing UI  
- [NEXT] Rich timeline zoom behaviours  
- [NEXT] Skinnable themes  
- [NEXT] Customisable keyboard shortcuts  
- [ADVANCED] Context-aware UI surfaces (auto-switch editing tools)

## 20.2 Accessibility  
- [CORE] Scalable UI  
- [NEXT] Screenreader support  
- [ADVANCED] High-contrast & dyslexia-friendly modes  

---

# End of Inbox
This inbox is intentionally comprehensive.  
Items will be moved into architecture documents as they graduate from “idea” to “specification”.  
Anything implemented must originate from, or be distilled into, an item here.
