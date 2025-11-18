# Scripting & Extensibility Architecture

This document defines the architecture for **scripting, automation,
extensibility and custom tooling** in Loophole.

The goal is to enable deep automation and custom workflows without compromising:
- realtime safety,
- engine stability,
- project integrity,
- cross-platform behaviour.

Scripting must feel like a natural part of Loophole while remaining strictly
sandboxed and isolated from engine internals.

This architecture builds on:

- `03-signal.md`
- `04-aura.md`
- `05-composer.md`
- `10-parameters.md`
- `11-node-graph.md`
- `14-timebase-tempo-and-groove.md`
- `17-automation-and-modulation.md`
- `18-midi-architecture.md`
- `21-control-surfaces-and-assistive-hardware.md`
- IPC specifications (Pulse↔Aura↔Signal)

---

# 1. Goals

Loophole scripting must:

1. **Be safe and isolated**
   - Scripts must never crash the audio engine or Pulse.
   - Scripts cannot block realtime threads.
   - Scripts run with limited permissions in a sandbox.

2. **Be powerful**
   - Access to the full project model (Pulse API).
   - Respond to user actions, timelines, hardware events.
   - Manipulate tracks, clips, parameters, scenes, Nodes.
   - Build custom UIs in Aura.
   - Automate workflows (render pipelines, batch edits, comping, asset management).

3. **Be cross-platform and deterministic**
   - Same behaviour on macOS, Windows, Linux.
   - Scripts should not depend on local filesystem layouts or machine-specific details.

4. **Be long-term maintainable**
   - Strongly versioned API surface.
   - Backwards compatibility where appropriate.
   - Scripts packaged in a way Composer can distribute curated “official scripts”.

---

# 2. Language Choice & Execution Environment

## 2.1 Choice of Scripting Language

Loophole supports *multiple* scripting languages eventually, but the **first and primary language is JavaScript/TypeScript**, hosted inside:

- a fully sandboxed V8 or QuickJS isolate  
- running **inside Pulse** (not Signal or Aura)

Reasons:

- secure, well-understood sandboxing,
- runs well cross-platform,
- predictable performance,
- integrates with structured IPC and JSON schemas,
- easy to implement restrictions (no direct file I/O, no blocking operations).

## 2.2 Sandbox Architecture

The sandbox has:

- no filesystem access (unless Pulse exposes virtual file APIs),
- no network access,
- no ability to block the Pulse event loop,
- no eval outside allowed code,
- timeouts to prevent infinite loops,
- memory limits per script.

Pulse exposes a *single* host object:

```
loophole: LoopholeAPI
```

Scripts cannot see any other global bindings.

---

# 3. Script Lifecycle

Scripts can be:

- **Session Scripts**  
  Loaded with a project; persist only while the project is open.

- **Global Scripts**  
  Installed by the user; available across all projects.

- **Packaged Extensions**  
  Bundles of scripts + UI + metadata + Composer integration.

Each script has:

```
ScriptMetadata {
  id;                    // unique, namespaced, e.g. "user.clipTools"
  name;
  version;
  description;
  permissions[];         // declared capabilities
  uiEntryPoints[];       // optional custom UI views
  triggers[];            // event-based triggers
}
```

Pulse manages:

- script loading,
- sandbox initialisation,
- lifecycle events,
- IPC boundaries.

---

# 4. Permissions Model

Scripts must declare permissions:

```
permissions: [
  "project.read",
  "project.write",
  "tracks.read",
  "tracks.write",
  "nodes.read",
  "nodes.write",
  "clips.read",
  "clips.write",
  "parameters.read",
  "parameters.write",
  "hardware.listen",
  "hardware.control",
  "ui.render",
  "filesystem.virtual",
  "render.offline",
  "render.ghost",
  "automation.modify"
]
```

Pulse enforces permissions at the API call boundary.

No permission = no access.

---

# 5. Trigger Model

Scripts respond to events.

### 5.1 Project Events
- onProjectLoad  
- onProjectSave  
- onProjectClose  
- onProjectChanged  

### 5.2 Track/Clip Events
- onTrackAdded  
- onClipCreated  
- onClipModified  
- onClipPipelineChanged  
- onCompStateChanged  

### 5.3 Timeline & Transport Events
- onPlay  
- onStop  
- onRecord  
- onTimelinePosition  

### 5.4 Parameter Events
- onParameterChanged  
- onGhostRendered  

### 5.5 Hardware Events
- onControlSurfaceInput  
- onMidiMessage  
- onDeviceConnected  

### 5.6 Diagnostics Events
- onBufferUnderrun  
- onPluginCrash  
- onNodeOverload  

Scripts register handlers:

```ts
loophole.on('onClipCreated', (clip) => {
    // ...
});
```

Pulse throttles and debounces events to guarantee isolation.

---

# 6. The Scripting API Surface

An *abbreviated* overview of the Loophole API:

```
LoopholeAPI {
  project: ProjectAPI
  tracks: TracksAPI
  clips: ClipsAPI
  nodes: NodeGraphAPI
  parameters: ParameterAPI
  automation: AutomationAPI
  mixer: MixerAPI
  launcher: LauncherAPI
  hardware: HardwareAPI
  events: EventAPI
  ui: UIAPI
  render: RenderAPI
  diagnostics: DiagnosticsAPI
}
```

### 6.1 Read/Write Boundaries

**Pulse is the source of truth.**  
The scripting API **never** directly touches Signal or Aura.

Scripts make *commands* to Pulse; Pulse updates the model; Aura & Signal respond downstream via IPC.

---

# 7. Custom UI: Script Panels in Aura

Scripts can declare:

```
uiEntryPoints: [
  { id: "clipTools", type: "panel", title: "Clip Tools" }
]
```

Aura exposes:

- a sidebar for script panels,
- modals/dialogs,
- custom canvas regions,
- grid-based control surfaces for rich UIs.

Custom UI code runs in Aura (in a safe iframe-like sandbox) and communicates with the script sandbox via Pulse.

UI layer uses:

- message-based protocol,
- structured JSON,
- no direct DOM access to Aura root tree.

---

# 8. Packaging & Distribution

Extensions are packaged as:

```
.loophole-extension.zip
```

Containing:

- scripts (JS/TS compiled),
- metadata (manifest.json),
- assets (icons, UI),
- optional Composer descriptors.

Composer supports:

- extension discovery,
- rating,
- popularity stats,
- update delivery,
- fingerprinting unsafe or broken extensions.

Pulse validates signatures before loading.

---

# 9. Integration with Composer

Composer contributes:

- recommended mappings for control-surface scripts (e.g. drum-pad sequencer tools),
- statistical insight into script usage patterns,
- QA metadata:
  - "This script is known to slow down large projects"
  - "This extension modifies automation in heavy patterns"

Composer may also suggest scripts based on project context:
- “You’re editing drums — consider installing Drum Patterns Toolkit.”
- “This project has frequent node overloads — Freeze Assistant script recommended.”

Telemetry is opt-in.

---

# 10. Engine Constraints & Safety

Scripts must **never** run on a realtime thread.

Pulse enforces:

- max execution time per call,
- cooperative scheduling,
- API-level throttling for large loops,
- streaming APIs for big data (automation curves, clip lists),
- auto-termination if script misbehaves.

Scripts cannot:
- run arbitrarily long loops,
- allocate unbounded memory,
- block rendering,
- spawn threads,
- perform unsafe recursion.

---

# 11. Extensibility Beyond Scripts

Future extensibility mechanisms that plug into this architecture:

### 11.1 Native Extensions (Rust / C++)
With strict boundaries:
- not allowed on realtime threads,
- cannot provide realtime DSP (that's reserved for Node types),
- isolated process model,
- share the same permissions manifesto.

### 11.2 Plugin Extensions
Tools that integrate at Pulse level to extend data model handling (e.g. custom comping algorithms, metadata processors).

### 11.3 Asset & Library Extensions
Custom media importers, metadata generators, tagging systems.

---

# 12. Examples of Future Script Use Cases

1. Batch track renamer  
2. Smart arranger macros  
3. Scene randomiser  
4. Clip generator for drums  
5. ARA batch render tools  
6. Ghost render validators  
7. Plugin preset randomisers  
8. Control surface alternate modes  
9. Diagnostics monitors  
10. Custom comping heuristics  
11. Track versioning automation  
12. Custom bounce/render workflows  
13. Tagging automation for media library  
14. MIDI transformation tools  

Everything is sandboxed, reversible, opt-in, and doesn’t risk project corruption.

---

# 13. Summary

This scripting architecture ensures:

- a **safe, fully sandboxed** execution layer,
- access to a deep and powerful model API,
- perfect integration with Pulse → Aura → Signal,
- custom UI capabilities,
- trigger- and event-driven automation,
- Composer-assisted intelligence,
- safe extensibility for power users without destabilisation.

Loophole becomes scriptable in a controlled, elegant and extendable fashion,
allowing a thriving ecosystem while keeping the engine rock-solid.
