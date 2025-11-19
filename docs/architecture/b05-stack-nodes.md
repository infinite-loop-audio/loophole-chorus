# StackNode Architecture

## Contents

- [1. Overview](#1-overview)
- [2. Problem Space](#2-problem-space)
- [3. StackNode: Definition](#3-stacknode-definition)
- [4. Instantiation Triggers](#4-instantiation-triggers)
- [5. Plugin Variant Lifecycle](#5-plugin-variant-lifecycle)
  - [5.1 State Machine](#51-state-machine)
  - [5.2 Loading Rules](#52-loading-rules)
  - [5.3 Automation Semantics](#53-automation-semantics)
- [6. Behaviour Within the Node Graph](#6-behaviour-within-the-node-graph)
- [7. Handling Missing Plugins](#7-handling-missing-plugins)
- [8. IPC Considerations](#8-ipc-considerations)
  - [8.1 Pulse Internal Representation](#81-pulse-internal-representation)
  - [8.2 Pulse → Signal Transfer](#82-pulse--signal-transfer)
  - [8.3 Aura Integration](#83-aura-integration)
  - [8.4 Composer Integration](#84-composer-integration)
- [9. User Experience Principles](#9-user-experience-principles)
- [10. Future Extensions](#10-future-extensions)
- [11. Constraints](#11-constraints)

---

## 1. Overview

A **StackNode** is a first-class Node type in Loophole’s processing architecture
that contains **multiple interchangeable plugin variants** while presenting as a
single logical processor in the channel graph.

Only one plugin in the stack is *active* at any time; switching variants does
not change the node graph shape, does not break automation, and does not alter
routing or CPU topology.

StackNodes:

- enable **A/B testing** of plugins,
- support **lossless fallback behaviour** for missing plugins,
- preserve all plugin automation/state across variants,
- fit naturally within the Node model,
- avoid destructive project edits when auditioning replacements.

---

## 2. Problem Space

Existing DAWs treat plugin replacement as a destructive operation. This causes:

- loss of parameter automation,
- complicated copy/paste workflows,
- fragile handling of missing plugins,
- inability to maintain non-destructive alternatives,
- cumbersome plugin A/B comparison.

Loophole solves this by abstracting plugin *identity* from plugin *position*.

A StackNode acts as a stable container; its child variants may come and go.

---

## 3. StackNode: Definition

A StackNode is a concrete Node type defined as:

```
StackNode
{
    id: NodeId,
    variants: [PluginVariant],
    active_index: usize
}
```

Where:

- `PluginVariant` contains:
  - plugin metadata,
  - plugin type (VST3/CLAP/AU/etc),
  - serialised plugin state,
  - automation lane bindings,
  - missing/errored state flags.

StackNodes appear in the channel’s Node chain like any other processor node:

```
[InstrumentNode] → [EQ StackNode] → [CompressorNode] → [SendNode]
```

The rest of the system treats the StackNode as a single effect/instrument node.

---

## 4. Instantiation Triggers

A StackNode is **not** used by default. It is instantiated through one of three
mechanisms:

### 4.1 Automatically when a plugin is missing during project load
Pulse detects a plugin that cannot be instantiated (missing, version mismatch,
failing integrity check) and wraps it in a StackNode containing:

- Variant 0: missing/original plugin (“unavailable” state)
- Variant 1: optional suggested replacement (if Pulse/Composer deem appropriate)

Aura presents this clearly to the user, preserving the original variant.

### 4.2 User initiates A/B testing
From the plugin UI in Aura, the user clicks **“A/B switch mode”**.  
Pulse converts the existing PluginNode into a StackNode containing:

- Variant 0: original plugin
- Variant 1: empty slot

User adds variants freely.

### 4.3 User explicitly creates a StackNode
Aura exposes a “Create StackNode” action independent of plugin loading.

User chooses:
- StackNode type: “Effect” or “Instrument”
- Initial empty variant slots

---

## 5. Plugin Variant Lifecycle

### 5.1 State Machine

Each variant uses a shared four-stage lifecycle for CPU/memory management:

```
Idle → Engaged → Cooling → Idle
```

#### Idle
- Only active variant is fully instantiated.
- All other variants unloaded: no CPU, minimal RAM.

#### Engaged
Triggered when:
- the StackNode UI is open,
- user hovers or selects variants,
- user triggers rapid switching (A/B mode).

Behaviour:
- All variants are loaded concurrently.
- Switching is instant.

#### Cooling
After user stops interacting (timeout ~2–5s):
- Inactive variants stop DSP processing but retain state in memory.

#### Idle (again)
After a longer timeout or end of playback:
- Inactive variants are fully unloaded.

### 5.2 Loading Rules

- Signal may pre-allocate memory for multiple variants in Engaged mode.
- Only one variant receives audio and MIDI at a time.
- Variants do not share parameter values unless explicitly mapped.

### 5.3 Automation Semantics

Automation belongs to the **variant**, not the StackNode.

- Each plugin variant maintains its own automation set.
- When switching active variant, the corresponding variant’s automation becomes active.
- Automation lanes linked to inactive variants remain dormant.
- Automation never blends or merges across variants.

---

## 6. Behaviour Within the Node Graph

StackNodes behave identically to any other processor node:

- appear as a single node in processing order,
- same bypass rules,
- same routing semantics,
- same parameter exposure model,
- same latency reporting behaviour.

Swapping the active variant is equivalent to swapping the underlying processor
while keeping node identity and routing stable.

---

## 7. Handling Missing Plugins

When loading a project:

1. Pulse identifies missing plugins.
2. Instead of failing or deleting nodes, Pulse wraps each missing instance in a
   StackNode.
3. The missing variant receives:
   - original plugin ID,
   - saved state (untouched),
   - an explicit `variant.missing = true`.

Users may:

- add alternative plugin variants,
- keep working seamlessly,
- re-enable the original plugin when it becomes available.

Composer may provide recommendations for alternatives.

---

## 8. IPC Considerations

### 8.1 Pulse Internal Representation

Pulse stores a StackNode as:

```
{
  nodeId,
  nodeKind: "stack",
  variants: [
    {
      variantId,
      pluginIdentifier,
      pluginFormat,
      stateBlob,
      missing: bool,
      metadata: { … }
    }
  ],
  activeIndex
}
```

### 8.2 Pulse → Signal Transfer

Pulse sends full variant metadata to Signal, but Signal:

- only loads the active variant under normal conditions,
- loads all variants when Pulse instructs (Engaged),
- unloads variants when requested (Cooling/Idle),
- exposes events for:
  - variant loading/unloading,
  - variant activation,
  - missing plugin placeholder creation.

### 8.3 Aura Integration

Aura displays StackNodes as:

- a single tile in the Node chain,
- internally showing:
  - variant list,
  - active variant,
  - missing variant indicators,
  - add/remove variant buttons,
  - A/B mode buttons.

Aura may allow:
- instant switching,
- soft “hover preview” (optional),
- variant ordering.

### 8.4 Composer Integration

Composer assists via:

- suggesting replacements for missing plugins,
- ranking alternatives by usage statistics,
- highlighting community-preferred mappings,
- providing normalised parameter name mapping if variants differ structurally.

---

## 9. User Experience Principles

- Switching must be **instantaneous** (Engaged mode ensures this).
- Missing plugins must be **non-destructive** to data.
- Automation must remain **intact** per variant.
- Users should be encouraged to experiment without penalty.
- UI should clearly indicate:
  - which variant is active,
  - which variants are missing,
  - which variants are loaded/unloaded.

---

## 10. Future Extensions

- Cross-variant parameter morphing (safe, explicit, opt-in).
- Multi-variant parallel processing (turn StackNode into SplitNode extension).
- Machine-learning recommendation engine (Composer).
- Plugin “profiles” to help map parameters between heterogeneous variants.
- StackNode templates (“Mastering stacks”, “Vocal chain variants”, etc.).

---

## 11. Constraints

- StackNodes must not break the deterministic rendering model.
- Switching variants must not alter downstream latency compensation behaviour.
- Inactive variants must never consume DSP in Idle/Cooling modes.
- StackNodes do not support mixing multiple active variants — that is the remit
  of a future SplitNode or GraphNode system.
- IPC spec must reflect both variant-level and node-level events.
