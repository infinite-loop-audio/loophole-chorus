# Signal IPC – Plugin Domain

This document defines the **Plugin** IPC domain between Pulse and Signal.

The Plugin domain is responsible for:

- discovering and describing available plugins (VST3, CLAP, AU, internal),
- instantiating and destroying plugin instances,
- querying plugin parameter and I/O metadata,
- getting/setting plugin state blobs (for save/load and migration),
- reporting plugin-level errors and determinism hints to Pulse.

Pulse owns:

- which plugins are used where (in Nodes),
- plugin aliasing and mapping for project stability (IDs, migrations),
- high-level categorisation and naming (with Composer’s help).

Signal owns:

- actual interaction with plugin SDKs and binaries,
- lifetime and internal handle management,
- enforcing real-time safety at the engine level.

Aura never talks to Signal directly; all commands flow **Pulse → Signal**.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Plugin domain:

- scans/queries the plugin inventory (as initiated by Pulse),
- creates plugin instances on request,
- destroys plugin instances,
- exposes:
  - parameter lists & properties,
  - program/preset support,
  - I/O configuration (channels, side-chains, MIDI),
- serialises & deserialises plugin state blobs,
- reports:
  - plugin crashes,
  - failed loads,
  - determinism hints (for anticipative scheduling).

### 1.2 Non-Goals

The Plugin domain does **not**:

- manage Node placement in the graph (`signal.graph`),
- manage Node parameter values directly (that’s the parameter/automation/gesture
  domains),
- render or position plugin UIs (`signal.pluginUI` / Pulse plugin-UI domain),
- decide how plugins are presented in the UI (that’s Aura & Composer).

---

## 2. Identity & Handle Model

### 2.1 Plugin Descriptors

A **Plugin Descriptor** identifies a specific plugin **type**:

```json
{
  "pluginId": "plugin:vst3:FabFilter:ProQ3",
  "format": "vst3",
  "vendor": "FabFilter",
  "name": "Pro-Q 3",
  "version": "3.12.0",
  "categories": ["EQ", "Filter"],
  "isInstrument": false
}
```

- `pluginId` is assigned by Pulse/Signal jointly, based on discovery.
- Pulse maintains a mapping between plugin IDs and user-friendly aliases.

### 2.2 Plugin Instances

A **Plugin Instance** is a specific live instance of a plugin type:

```json
{
  "pluginInstanceId": "pluginInstance:abc123",
  "pluginId": "plugin:vst3:FabFilter:ProQ3"
}
```

- Instances are referenced by Nodes via `pluginInstanceId` in the Graph domain.
- Multiple instances can share the same `pluginId`.

### 2.3 StackNode Variants

StackNodes (defined in the Graph domain) contain multiple plugin variants. From the Plugin domain perspective:

- Each variant within a StackNode may have its own `pluginInstanceId` (if the variant is instantiated).
- Signal typically instantiates only the **active variant** under normal conditions.
- When Pulse instructs Signal to enter "Engaged" mode (e.g. during A/B switching or when the StackNode UI is open), Signal may load all variants concurrently.
- Signal must support loading and unloading individual variants on request from Pulse.
- Variants with `missing: true` are not instantiated; Signal creates placeholders only and does not attempt plugin loading.
- StackNode variant switching must maintain deterministic processing and must not break latency reporting for the node (switching should occur on buffer boundaries).

---

## 3. Message Flow Overview

Typical flows:

1. At startup or on demand, Pulse calls `plugin.listAvailable` or
   `plugin.rescan` to discover plugins.
2. When a Node of kind `PluginNode` or `StackNode` is created in Pulse:
   - For PluginNodes: Pulse calls `plugin.createInstance`, gets back a `pluginInstanceId`, puts that ID into the Node's config, then applies it via `signal.graph`.
   - For StackNodes: Pulse creates plugin instances for each variant (typically only the active variant initially), associates `pluginInstanceId`s with variant indices, and applies the StackNode via `signal.graph` with variant metadata.
3. When saving/loading projects:
   - Pulse fetches plugin state blobs via `plugin.getState`,
   - stores them in the project model,
   - later calls `plugin.setState` to restore.
4. When plugins crash or misbehave:
   - Signal emits `plugin.crashed` or `plugin.unstable`.

---

## 4. Commands (Pulse → Signal)

### 4.1 `plugin.listAvailable`

Request a list of known/available plugins.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "list-available-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "listAvailable",
  "priority": "normal",
  "payload": {
    "filters": {
      "format": null,          // "vst3" | "clap" | "au" | null
      "isInstrument": null     // true | false | null
    }
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "list-available-response-456",
  "cid": "list-available-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "listAvailable",
  "priority": "normal",
  "payload": {
    "plugins": [
      {
        "pluginId": "plugin:vst3:FabFilter:ProQ3",
        "format": "vst3",
        "vendor": "FabFilter",
        "name": "Pro-Q 3",
        "version": "3.12.0",
        "categories": ["EQ", "Filter"],
        "isInstrument": false
      }
    ]
  },
  "error": null
}
```

Semantics:

- Pulse can cache this list and combine it with Composer metadata.
- `filters` are optional; Signal can ignore them if unsupported.

---

### 4.2 `plugin.rescan`

Request a plugin rescan (full or partial).

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "rescan-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "rescan",
  "priority": "normal",
  "payload": {
    "options": {
      "formats": ["vst3", "clap"],
      "force": false
    }
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "rescan-response-456",
  "cid": "rescan-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "rescan",
  "priority": "normal",
  "payload": {
    "scanId": "pluginScan:2025-11-18T12:00:00Z"
  },
  "error": null
}
```

Semantics:

- Scanning may be long-running and performed in the background.
- Results (new/removed/updated plugins) are delivered via events:
  - `plugin.scanCompleted`,
  - `plugin.added`,
  - `plugin.removed`,
  - `plugin.updated`.

---

### 4.3 `plugin.createInstance`

Create a new instance of a plugin.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "create-instance-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "createInstance",
  "priority": "high",
  "payload": {
    "pluginId": "plugin:vst3:FabFilter:ProQ3",
    "options": {
      "initialPreset": null,
      "deterministicHint": true
    }
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "create-instance-response-456",
  "cid": "create-instance-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "createInstance",
  "priority": "high",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "pluginId": "plugin:vst3:FabFilter:ProQ3"
  },
  "error": null
}
```

Semantics:

- The newly created instance is not attached to any Channel/Node yet.
- Pulse must integrate the `pluginInstanceId` into its Node model, then apply
  via `graph.applyDelta` or snapshot.
- `deterministicHint` gives Signal a hint:
  - `true` if Pulse believes this plugin is deterministic (safe for anticipative
    rendering),
  - `false` otherwise.
- Signal can refine the determinism assessment over time and report back via
  telemetry/diagnostics.

---

### 4.4 `plugin.destroyInstance`

Destroy a plugin instance that is no longer needed.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "destroy-instance-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "destroyInstance",
  "priority": "high",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "destroy-instance-response-456",
  "cid": "destroy-instance-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "destroyInstance",
  "priority": "high",
  "payload": {},
  "error": null
}
```

Semantics:

- Signal must ensure that the instance is no longer being used in the graph
  before destroying it.
- If the instance is still in use, an error is returned.

---

### 4.5 `plugin.getInfo`

Request extended information about a plugin type.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "get-info-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "getInfo",
  "priority": "normal",
  "payload": {
    "pluginId": "plugin:vst3:FabFilter:ProQ3"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "get-info-response-456",
  "cid": "get-info-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "getInfo",
  "priority": "normal",
  "payload": {
    "pluginId": "plugin:vst3:FabFilter:ProQ3",
    "format": "vst3",
    "vendor": "FabFilter",
    "name": "Pro-Q 3",
    "version": "3.12.0",
    "categories": ["EQ", "Filter"],
    "isInstrument": false,
    "io": {
      "maxInputs": 10,
      "maxOutputs": 10,
      "supportsSidechain": true
    },
    "capabilities": {
      "supportsDoublePrecision": true,
      "supportsMIDI": true,
      "supportsMPE": false
    }
  },
  "error": null
}
```

Semantics:

- Pulse uses this to:
  - validate routing,
  - present correct UI,
  - determine suitability for certain tasks.

---

### 4.6 `plugin.getParameters`

Request the parameter list for a plugin instance (or type, if supported).

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "get-parameters-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "getParameters",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "get-parameters-response-456",
  "cid": "get-parameters-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "getParameters",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "parameters": [
      {
        "id": "param:cutoff",
        "name": "Cutoff",
        "unit": "Hz",
        "min": 20.0,
        "max": 20000.0,
        "default": 1000.0,
        "automationSafe": true,
        "stepped": false,
        "kind": "float" // "float" | "int" | "bool" | "enum"
      }
    ]
  },
  "error": null
}
```

Semantics:

- Pulse maps these to its parameter model, allowing:
  - automation lane binding,
  - normalised/enumerated UI control,
  - aliasing between plugin versions.

---

### 4.7 `plugin.getPrograms`

Request program/preset list for a plugin instance (if supported).

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "get-programs-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "getPrograms",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "get-programs-response-456",
  "cid": "get-programs-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "getPrograms",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "programs": [
      {
        "index": 0,
        "name": "Init"
      },
      {
        "index": 1,
        "name": "Bright EQ"
      }
    ]
  },
  "error": null
}
```

Semantics:

- Program support varies by plugin and format; Signal may return an empty list.

---

### 4.8 `plugin.setProgram`

Set the current program for a plugin instance.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "set-program-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "setProgram",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "index": 1
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "set-program-response-456",
  "cid": "set-program-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "setProgram",
  "priority": "normal",
  "payload": {},
  "error": null
}
```

Semantics:

- May internally update plugin state; Pulse may want to fetch new state via
  `plugin.getState` after program changes if it stores full state blobs.

---

### 4.9 `plugin.getState`

Get the serialised state blob for a plugin instance.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "get-state-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "getState",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "get-state-response-456",
  "cid": "get-state-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "getState",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "state": {
      "format": "opaque",
      "dataBase64": "AAECAwQFBgcICQoL..." 
    }
  },
  "error": null
}
```

Semantics:

- `dataBase64` is an opaque blob; Pulse stores it in the project.
- Format is typically host-specific plugin state format (VST chunk, CLAP state,
  AU preset, etc.).

---

### 4.10 `plugin.setState`

Set the serialised state blob for a plugin instance.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "set-state-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "setState",
  "priority": "normal",
  "payload": {
    "pluginInstanceId": "pluginInstance:abc123",
    "state": {
      "format": "opaque",
      "dataBase64": "AAECAwQFBgcICQoL..." 
    },
    "options": {
      "allowPartial": true
    }
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "set-state-response-456",
  "cid": "set-state-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "setState",
  "priority": "normal",
  "payload": {},
  "error": null
}
```

Semantics:

- Used on project load or migration.
- `allowPartial` indicates whether the host may apply partial state when a
  perfect match is impossible (implementation-specific behaviour).

---

### 4.11 `plugin.assessDeterminism`

Request an explicit determinism assessment for a plugin type or instance.

**Command Envelope (Pulse → Signal):**

```jsonc
{
  "v": 1,
  "id": "assess-determinism-123",
  "cid": null,
  "ts": "2025-11-20T12:34:56.789Z",
  "origin": "pulse",
  "target": "signal",
  "domain": "plugin",
  "kind": "command",
  "name": "assessDeterminism",
  "priority": "normal",
  "payload": {
    "pluginId": "plugin:vst3:FabFilter:ProQ3"
  },
  "error": null
}
```

**Response Envelope (Signal → Pulse):**

```jsonc
{
  "v": 1,
  "id": "assess-determinism-response-456",
  "cid": "assess-determinism-123",
  "ts": "2025-11-20T12:34:56.790Z",
  "origin": "signal",
  "target": "pulse",
  "domain": "plugin",
  "kind": "response",
  "name": "assessDeterminism",
  "priority": "normal",
  "payload": {
    "pluginId": "plugin:vst3:FabFilter:ProQ3",
    "deterministic": true,
    "notes": "No known time-dependent or random modulation internally."
  },
  "error": null
}
```

Semantics:

- Used together with Composer’s telemetry to decide:
  - cohort placement,
  - anticipative rendering eligibility.
- Actual assessment may be partial and heuristic.

---

## 5. Events (Signal → Pulse)

### 5.1 `plugin.scanCompleted`

Plugin scan has completed.

**Payload**

```json
{
  "event": "plugin.scanCompleted",
  "scanId": "pluginScan:2025-11-18T12:00:00Z",
  "summary": {
    "added": 10,
    "removed": 2,
    "updated": 3
  }
}
```

---

### 5.2 `plugin.added`

A new plugin type has been discovered.

**Payload**

```json
{
  "event": "plugin.added",
  "plugin": {
    "pluginId": "plugin:vst3:NewVendor:NewPlugin",
    "format": "vst3",
    "vendor": "NewVendor",
    "name": "NewPlugin",
    "version": "1.0.0",
    "categories": [],
    "isInstrument": false
  }
}
```

---

### 5.3 `plugin.removed`

A plugin type is no longer available.

**Payload**

```json
{
  "event": "plugin.removed",
  "pluginId": "plugin:vst3:OldVendor:OldPlugin"
}
```

Pulse should:

- mark this plugin as unavailable,
- consult Composer and user mappings to propose replacements.

---

### 5.4 `plugin.updated`

Plugin metadata or binary has changed.

**Payload**

```json
{
  "event": "plugin.updated",
  "pluginId": "plugin:vst3:FabFilter:ProQ3",
  "oldVersion": "3.11.0",
  "newVersion": "3.12.0"
}
```

Pulse and Composer may:

- inspect parameter mappings,
- suggest migrations,
- adjust aliases.

---

### 5.5 `plugin.crashed`

A plugin instance has crashed or misbehaved severely.

**Payload**

```json
{
  "event": "plugin.crashed",
  "pluginInstanceId": "pluginInstance:abc123",
  "pluginId": "plugin:vst3:SomeVendor:UnstableFX",
  "message": "Plugin caused an access violation during processing.",
  "details": {
    "duringCallback": true
  }
}
```

Semantics:

- Signal should isolate the crash as much as possible:
  - mute or bypass the Node,
  - keep engine alive if feasible.
- Pulse should:
  - notify the user,
  - mark the plugin as unstable for this session,
  - potentially suggest alternatives.

---

### 5.6 `plugin.warning`

Non-fatal plugin issues.

**Payload**

```json
{
  "event": "plugin.warning",
  "pluginInstanceId": "pluginInstance:def456",
  "pluginId": "plugin:vst3:SomeVendor:SlowFX",
  "code": "highCpuUsage",
  "message": "Plugin is consistently using high CPU.",
  "details": {
    "averageLoadPercent": 60.0
  }
}
```

Pulse and Diagnostics can display this in performance panels.

---

## 6. Error Handling

Plugin commands can fail for reasons including:

- invalid `pluginId` or `pluginInstanceId`,
- plugin binary missing or incompatible,
- plugin refusing state blob,
- running out of internal resources (instance limit, memory).

Example error:

```json
{
  "error": {
    "code": "pluginNotFound",
    "message": "No plugin with ID plugin:vst3:FabFilter:ProQ3 was found.",
    "domain": "plugin",
    "details": {
      "pluginId": "plugin:vst3:FabFilter:ProQ3"
    }
  }
}
```

Pulse must treat these as authoritative and adjust user-visible state and
mappings accordingly.

---

## 7. Relationship to Other Signal Domains

- **signal.graph**  
  Plugin-backed Nodes (PluginNode, InstrumentNode) require a `pluginInstanceId`
  created via this domain. Graph defines where plugin instances live in the
  signal chain.

- **signal.media**  
  Some plugins may use media data (e.g. IR reverb loaders); how they access it
  is plugin-specific and usually encapsulated inside the plugin.

- **signal.engine & signal.diagnostics**  
  Engine controls global processing; Diagnostics reports plugin-level CPU load
  and stability. Plugin domain provides metadata and state, not performance
  metrics directly.

- **signal.pluginUI** (to be defined separately)  
  Plugin UI domain uses the same `pluginInstanceId` for window management and
  UI embedding.

---

## 8. Extensibility

Possible future additions:

- explicit migration APIs (mapping parameters between different plugin versions
  or different plugins for project portability),
- plugin sandboxing controls (per-plugin isolation flags),
- per-plugin logging/trace controls for debugging,
- parameter group metadata for better UI layout and automation grouping.

All extensions should remain compatible with the core model:

- stable `pluginId`/`pluginInstanceId` identity,
- separation of metadata, state, and processing graph.
