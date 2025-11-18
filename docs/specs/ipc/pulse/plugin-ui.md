# Pulse Plugin UI Domain Specification

This document defines the Plugin UI domain of the Pulse project protocol.  
It covers commands and events for managing:

- plugin editor windows,
- their position/size across multi-screen setups,
- opening/closing plugin UIs,
- focus/raise behaviour,
- geometry synchronisation between Aura and Signal,
- plugin-initiated resizes.

This domain works alongside the Window Layout Architecture to ensure:

- deterministic window placement,
- user layouts recall correctly per ScreenSet and per machine,
- plugin windows never appear off-screen,
- plugins can resize their own UI safely,
- Aura retains full control of UI placement decisions.

Pulse acts as the coordinator between Aura and Signal.  
Signal owns native plugin window creation and destruction.  
Aura owns layout decisions and persistence.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Window Identity and Keys](#2-window-identity-and-keys)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Window Lifecycle](#31-window-lifecycle)  
  - [3.2 Window Geometry and State](#32-window-geometry-and-state)  
  - [3.3 Focus and Z-Order](#33-focus-and-z-order)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Lifecycle Events](#41-lifecycle-events)  
  - [4.2 Geometry Events](#42-geometry-events)  
  - [4.3 Focus Events](#43-focus-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Relationship to Window Layout Architecture](#7-relationship-to-window-layout-architecture)

---

## 1. Overview

A plugin UI window:

- is created by **Signal**, because Signal loads and hosts the plugin,
- is positioned and sized by **Aura**, because the UI layer owns layout,
- is coordinated by **Pulse**, which forwards commands and events between them.

This domain defines:

- a consistent way to open/close plugin editors,
- a safe strategy for multi-screen placement,
- support for plugin-driven resize events,
- coordination of focus/raise behaviour.

Plugin UI windows are always **top-level native windows**, never embedded in Electron, to avoid blocking the audio thread and to ensure plugin compatibility.

---

## 2. Window Identity and Keys

Each plugin UI has a stable logical key:

```
plugin-ui:[trackId]:[channelId]:[nodeId]
```

Pulse assigns a unique `pluginUiId`.

Aura uses the logical key for:

- layout persistence,
- multi-screen profile management,
- grouping rules (e.g. opening several instances from the same plugin family).

Signal uses the numeric `pluginUiId` to manage native windows.

A plugin UI window is always associated with exactly one:

- `nodeId` (PluginNode),
- `channelId`,
- `trackId`.

---

## 3. Commands (Aura → Pulse)

### 3.1 Window Lifecycle

#### **`pluginUi.open`**
Request Signal to open the plugin UI for the given node.

Fields:
- `trackId`
- `channelId`
- `nodeId`
- optional `initialLayout` (rect + display ID, if Aura already knows where to put it)

Pulse:
- validates node existence,
- allocates `pluginUiId`,
- instructs Signal to create the native window,
- emits `pluginUi.opened` when ready.

---

#### **`pluginUi.close`**
Close the plugin UI.

Fields:
- `pluginUiId`

Signal destroys the native window and releases resources.

---

#### **`pluginUi.toggle`**
Convenience command (Aura may use this for UI shortcuts).

Fields:
- `trackId`, `channelId`, `nodeId`

Pulse determines whether the window is open and calls open or close accordingly.

---

### 3.2 Window Geometry and State

#### **`pluginUi.setGeometry`**
Set window position/size.

Fields:
- `pluginUiId`
- `x`, `y`, `width`, `height` (virtual desktop coordinates)
- optional `displayId`
- optional `mode` (`floating`, `maximised`, `fullscreen`)

This is initiated by Aura after applying ScreenSet layout rules.  
Signal must clamp geometry if the plugin tries to exceed screen size or minimum constraints.

---

#### **`pluginUi.requestResize`**
Instruct Signal to request a resize from the plugin (if supported).

Useful for resizing CLAP/UIs that support dynamic layouts.

Fields:
- `pluginUiId`
- `width`, `height` (requested content size)

If unsupported by the plugin, Signal reports failure.

---

#### **`pluginUi.refreshGeometry`**
Request current geometry from Signal.

Pulse forwards this to Signal; useful when windows are:

- moved by the OS window manager,
- resized by plugin,
- moved between displays.

Signal responds with `pluginUi.geometryChanged`.

---

### 3.3 Focus and Z-Order

#### **`pluginUi.raise`**
Bring the plugin UI window to front.

#### **`pluginUi.focus`**
Give the plugin window keyboard focus.

#### **`pluginUi.blur`**
Release focus.

Signal is responsible for interacting with the OS window system.

---

## 4. Events (Pulse → Aura)

### 4.1 Lifecycle Events

#### **`pluginUi.opened`**
Plugin UI ready.

Fields:
- `pluginUiId`
- `trackId`, `channelId`, `nodeId`
- plugin metadata (name, vendor)
- initial geometry (if available)

---

#### **`pluginUi.closed`**
Plugin UI closed (by user, plugin or Signal).

Fields:
- `pluginUiId`

Aura updates layout state accordingly.

---

### 4.2 Geometry Events

#### **`pluginUi.geometryChanged`**
Emitted when Signal detects:

- plugin UI was resized internally,
- user resized the OS window manually,
- OS changed display scale,
- plugin reflowed its UI.

Fields:
- `pluginUiId`
- updated rect (`x`, `y`, `width`, `height`)
- `displayId`
- optional constraints or plugin-reported preferred sizes

Aura must update its WindowDescriptor and persist the new geometry.

---

#### **`pluginUi.resizableChanged`**
Plugin reports new min/max size constraints.

Aura should store these and apply during next layout reconciliation.

---

### 4.3 Focus Events

#### **`pluginUi.focused`**
Signal reports plugin gained focus.

#### **`pluginUi.blurred`**
Signal reports plugin lost focus.

Useful for keyboard shortcut routing and input delegation rules.

---

## 5. Snapshot Semantics

Plugin UIs are **not persisted** as part of project snapshots.

What *is* stored in the project (per the Window Layout architecture):

- per-Node UI **open/closed flag** (optional),
- per-machine, per-ScreenSet WindowDescriptors for plugin windows.

On project load:

- Aura determines machineId + ScreenSet,
- retrieves plugin window descriptors,
- issues `pluginUi.open` and `pluginUi.setGeometry` as appropriate.

Pulse/IPCs do not include UI layout — that remains Aura’s responsibility.

---

## 6. Realtime and Cohort Considerations

Plugin UIs must **never** interfere with audio thread timing:

- Signal creates windows on a host UI thread, not the realtime audio thread.
- IPC commands for UI events (resize, focus) are processed asynchronously.
- Gesture streams (e.g. plugin parameter knobs) are handled separately and
  never via plugin UI IPC.

Plugin UI operations must not:

- block cohort transitions,
- block anticipative rendering,
- require synchronous parameter evaluation.

Plugin windows should not assume that parameter updates occur synchronously.

---

## 7. Relationship to Window Layout Architecture

This domain depends critically on:

**`24-windowing-and-plugin-ui.md`**, which defines:

- multi-screen awareness,
- WindowDescriptors,
- ScreenSet IDs,
- project-level layout persistence keyed by `(machineId, ScreenSetId)`,
- clamping rules to prevent off-screen windows.

Plugin UI IPC provides the **mechanism** to move windows;  
the Window Layout architecture provides the **policy** that decides *where* they go.
