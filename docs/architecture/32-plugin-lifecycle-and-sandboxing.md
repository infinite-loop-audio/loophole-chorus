# Plugin Lifecycle & Sandboxing Architecture

This document defines the complete **plugin lifecycle**, **sandboxing model**, and
**isolation semantics** for Loophole. It covers:

- plugin discovery and scanning,
- metadata normalisation (with Composer-assisted refinement),
- plugin identity and alias resolution,
- version & update handling,
- instance creation and destruction,
- crash containment and node-level isolation,
- deterministic vs non-deterministic classification,
- UI lifecycle and window management,
- format bridging (VST3, CLAP, AU),
- parameter mapping & persistence,
- safety constraints inside Signal,
- IPC responsibilities between Pulse ↔ Signal ↔ Aura.

It complements:

- `11-node-graph.md` (Nodes represent plugin processors)
- `12-mixer-and-channel-architecture.md`
- `14-channel-routing-and-send-return.md`
- `06-processing-cohorts-and-anticipative-rendering.md`
- `23-diagnostics-and-performance.md`
- `25-control-surfaces-and-assistive-hardware.md`
- Composer architecture (`05-composer.md`)
- Pulse/Signal IPC in `node.md`, `processor.md`, `hardware.md`

Plugins in Loophole are treated as first-class modular units, each running in an
isolated, crash-safe sandbox with stable identity and robust parameter mapping.

---

# 1. Goals

Loophole’s plugin system must:

1. **Be stable and crash-resistant**
   - A plugin crash must *never* crash Signal.
   - A plugin crash must *never* drop audio globally.
   - Crashed plugins can be restarted transparently.

2. **Provide predictable, deterministic behaviour where possible**
   - Distinguish deterministic from non-deterministic plugins.
   - Use this classification for anticipative rendering rules.

3. **Handle plugin versioning gracefully**
   - Project-safe identity mapping.
   - Composer-assisted alias maps.
   - Easy recovery when plugins update or move.

4. **Allow completely flexible routing**
   - Nodes can contain plugin instances.
   - Plugins can run anywhere in the NodeGraph.
   - Multiple instances of the same plugin may run concurrently.

5. **Provide a consistent UI lifecycle**
   - UI windows that can attach/detach cleanly.
   - Multi-screen awareness.
   - Reopen with persistent bounds per machine/display set.

6. **Support all major plugin formats**
   - VST3 (primary)
   - CLAP (primary)
   - AU (macOS)
   - Future formats possible via bridging layer.

7. **Be fast to load and responsive**
   - Hot-loading plugin UIs.
   - Prewarming states when needed for live triggering.

---

# 2. Plugin Identity & Metadata

Pulse stores detailed plugin metadata, normalised across formats.

```
PluginMeta {
  pluginId;                     // stable identity
  manufacturer;
  name;
  version;
  formats: [vst3, clap, au];
  pathPerFormat;                // file paths
  sandboxPolicy;                // strict | moderate | disabled (internal-only)
  deterministic: bool?;         // may be refined dynamically
  parameters: ParameterMeta[];
  ioConfig: IOConfiguration;
  features: {...};              // eg. supportsMPE, supportsARA, hasSidechain
  lastSeenVersion?;
  composerHints?;               // tag suggestions, parameter aliases, categorisation
}
```

Plugin IDs remain stable across:
- format changes (e.g. switching VST3 → CLAP),
- minor version updates,
- relocation of plugin bundles.

## 2.1 Composer Integration

Composer assists by:
- learning plugin categorisation across the user base,
- learning parameter alias mapping,
- detecting changes in plugin versions,
- suggesting fallback mappings if a plugin disappears,
- classifying deterministic vs non-deterministic behaviour based on observed data.

Pulse uses Composer as a “second opinion”, never a hard dependency.

---

# 3. Plugin Scanning

Signal performs scanning, but Pulse owns the canonical metadata.

### 3.1 Hybrid Scanning Model

1. **Initial Deep Scan** (first run)
   - Scan all plugin folders.
   - Load metadata, capabilities, categories.
   - Report deterministic heuristics.
   - Cache results.

2. **Incremental Scan** (normal startup)
   - Only re-scan changed or new plugin files.
   - Verify signatures, versions, timestamps.

3. **On-Demand Scan**
   - Triggered when user installs or removes plugins while Loophole is open.

Pulse reconciles scan results with plugin IDs and notifies Aura of:
- new plugins,
- updated plugins,
- missing plugins.

### 3.2 Crash-Safe Scanning

All scanning occurs in a dedicated **scanner sandbox** process:
- never loads plugin code into main Signal process,
- prevents malicious or unstable plugin crashes.

---

# 4. Plugin Lifecycle

## 4.1 Node Instantiation

When a Node of type `PluginNode` is created:

1. Aura → Pulse:  
   `node.create` with pluginId.
2. Pulse → Signal:  
   `plugin.instance.create(instanceId, pluginId, config)`.
3. Signal:
   - launches plugin in sandbox,
   - loads binary,
   - initialises parameters,
   - probes IO configuration,
   - reports ready.

## 4.2 Realtime-Safe Behaviour

Plugin instantiation **cannot block the realtime thread**.

Signal ensures:
- construction happens in worker threads,
- realtime thread only gets a pointer once plugin is ready,
- parameter/state restoration happens off the realtime thread.

## 4.3 Plugin State Restoration

Pulse serialises plugin state as:

```
PluginState {
  rawState;            // format-specific binary blob
  parameterValues;     // explicit overrides if available
  routingInfo?;
}
```

Upon project load:
- Pulse sends parameters first (safe),
- then sends raw binary state,
- then signals plugin to finalise setup.

This ensures:
- state is consistent,
- plugins can rebuild complex internal settings.

---

# 5. Sandboxing Model

Loophole uses **per-plugin-instance sandboxing**, not global sandboxing.

## 5.1 Sandboxing Options

```
sandboxPolicy:
  strict:    full process isolation per instance
  moderate:  shared process per plugin type, isolated threads
  disabled:  allowed only for trusted internal DSP
```

## 5.2 Strict Sandbox (Default)

Each plugin instance runs in its own process:

- crash kills *only* that sandbox process,
- Signal detects crash and:
  - replaces output with silence,
  - shows plugin crash indicator,
  - automatically attempts restart if safe.

## 5.3 Moderate Sandbox

Plugins that are known to be stable may run in a shared process, with:
- thread isolation,
- memory guards,
- stricter watchdog timers.

## 5.4 Disabled Sandboxing

Used for:
- internal DSP modules,
- verified high-stability plugins,
- testing.

Never default for third-party code.

---

# 6. Crash Detection & Recovery

Signal monitors:
- sandbox exit status,
- watchdog timers,
- realtime processing consistency,
- invalid buffer writes.

If a crash occurs:

### 6.1 Immediate Response
- plugin output replaced with silence,
- DSP graph continues uninterrupted,
- Pulse notified: `plugin.crashed(instanceId)`.

### 6.2 User-Safe Recovery
- Aura shows non-blocking crash badge,
- User can:
  - restart plugin instance,
  - open crash log,
  - replace plugin,
  - remove node.

### 6.3 Auto-Restart
Pulse may:
- attempt auto-restart if plugin supports state rehydration,
- load previous saved plugin state,
- resume processing.

Auto-restart is never done during a live performance unless user permits.

---

# 7. Deterministic vs Non-Deterministic Plugins

Pulse keeps a classification:

```
deterministic: true | false | unknown
```

Based on:
- plugin capabilities,
- measured consistency across runs,
- Composer’s statistical data,
- user overrides.

## 7.1 Why It Matters

Deterministic plugins:
- may run in anticipative cohorts,
- can be pre-rendered safely.

Non-deterministic plugins:
- must run in the realtime cohort,
- must be warmed before triggering,
- restricted during offline renders (e.g. repeated renders might differ).

## 7.2 Automatic Heuristics

Signal detects:
- whether output changes when parameters unchanged,
- whether plugin uses randomness,
- whether plugin fails to respond deterministically to silent input,
- heavy internal statefulness.

Pulse records heuristics, Composer aggregates global insights.

---

# 8. Plugin UI Lifecycle

## 8.1 UI Windows

Plugin UIs are:
- native plugin windows,
- embedded in Aura via window reparenting,
- or displayed as floating windows.

Window bounds are stored per:
- plugin instance,
- machine identifier,
- screen set (`docs/architecture/10-windowing-and-screen-management.md`).

## 8.2 UI Creation

When a UI is opened:
1. Aura requests UI from Pulse.
2. Pulse instructs Signal to create a UI session.
3. Signal opens plugin-provided editor window.
4. Aura repositions and parents the window appropriately.

## 8.3 UI Lifetime

UI state persists until:
- window destroyed by user,
- plugin instance destroyed,
- sandbox crash.

UI reopening must:
- restore bounds,
- reconnect event handlers,
- reattach to plugin instance.

---

# 9. Parameter Mapping & Persistence

Each plugin exposes:
- parameters with **stable internal IDs** when possible,
- category/subsection metadata if available,
- automation flags and ranges.

Pulse wraps these into:

```
ParameterMeta {
  parameterId;          // stable ID used by Loophole
  pluginInternalId;     // format-specific ID
  name;
  unit?;
  range;
  defaultValue;
  automationMode;
  smoothingPolicy?;
  composerAliases?;
}
```

### 9.1 Alias Resolution

When plugins update and change internal parameter IDs:

- Pulse attempts direct match,
- Composer provides statistical matches (e.g. renamed parameters across versions),
- User can resolve disagreements,
- Pulse stores resolved map in project.

### 9.2 Plugin Replacement

If plugin is missing:
- Pulse searches for alternatives:
  - same plugin in different format (VST3 ↔ CLAP),
  - same vendor, same name, different version,
  - Composer-suggested compatible plugins.

User decides whether to:
- replace,
- ignore,
- disable the node,
- freeze and flatten output.

---

# 10. Format Bridging

Loophole supports:

### 10.1 Native Formats
- VST3 (primary universal)
- CLAP (modern, preferred for advanced workflows)
- AU (macOS only)

### 10.2 Bridging Layer

Signal contains bridge loaders that:
- wrap plugin lifecycle APIs uniformly,
- expose feature flags (ARA, sidechain, MPE),
- ensure consistent parameter automation behaviour.

Bridge constraints:
- AU cannot be sandboxed strictly on some macOS versions; fallback to moderate.
- Future formats can be added with thin adapters.

---

# 11. NodeGraph Integration

A `PluginNode` integrates into the NodeGraph with:

```
PluginNode {
  nodeId;
  pluginInstanceId;
  parameterMap[];
  ioConfig;
  bypassState;
  oversampling?;
  latency;
}
```

### 11.1 Latency Management

Signal reports plugin latency:
- Pulse uses this for automatic delay compensation,
- zero-latency previews possible via bypass.

### 11.2 Dynamic IO Switching

Some plugins support:
- dynamic sidechain enabling,
- input channel changes,
- surround/object channel layouts (future).

Pulse validates graph changes and updates Signal accordingly.

---

# 12. Cohort Interaction

Plugins affect cohort assignment:

### Deterministic:
- safe for anticipative rendering.

### Non-deterministic:
- must run in realtime cohort,
- must be warmed before clip triggering,
- restricts anticipative lookahead windows.

Pulse updates cohort placement dynamically when:
- plugin added/removed,
- plugin changes mode,
- sandbox crash occurs.

---

# 13. Diagnostics & Logging

Signal reports:
- plugin load times,
- initialization errors,
- watchdog timeouts,
- DSP overruns,
- UI thread detachment,
- crash logs.

Pulse aggregates into:
- plugin health reports,
- stability metrics,
- per-instance crash frequency.

Aura displays:
- badge indicators,
- plugin stability warnings,
- recommendations from Composer.

---

# 14. IPC Responsibilities

### Aura → Pulse
- `plugin.browse`
- `plugin.openUI`
- `plugin.closeUI`
- `node.addPlugin`
- `node.removePlugin`
- `node.movePlugin`
- `plugin.replace`
- `plugin.parameter.set`
- `plugin.parameter.touch/untouch`
- `plugin.policy.setSandboxLevel`

### Pulse → Signal
- `plugin.instance.create`
- `plugin.instance.destroy`
- `plugin.instance.restart`
- `plugin.parameter.update`
- `plugin.ui.create`
- `plugin.ui.destroy`

### Signal → Pulse
- `plugin.ready`
- `plugin.crashed`
- `plugin.parameter.changed`
- `plugin.uiBoundsChanged`
- `plugin.latencyChanged`

---

# 15. Future Extensions

- Multi-sandbox GPU-enabled plugins  
- Machine-learning plugin profiles  
- Shared neural model execution nodes  
- Plugin-to-plugin communication buses  
- Network-hosted plugins (LAN / cloud)  
- Object-based processing formats (immersive audio)  
- Composer-driven plugin recommendations  

---

# 16. Summary

Loophole’s plugin architecture:

- provides fully sandboxed per-instance isolation,
- ensures crash-resistant realtime audio,
- allows flexible routing and NodeGraph integration,
- uses metadata, heuristics and Composer intelligence for plugin identity,
- handles versioning and aliasing safely,
- supports deterministic-aware cohort planning,
- exposes a clean IPC-driven lifecycle,
- supports complex UI lifecycles,
- is robust across VST3, CLAP and AU ecosystems.

This establishes a modern, powerful, and safe platform for plugin use in
professional audio production and performance.
