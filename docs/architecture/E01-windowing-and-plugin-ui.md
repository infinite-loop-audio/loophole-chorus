# Windowing & Plugin UI Architecture

This document describes the architecture for managing plugin UI windows and
Aura windows in a multi-screen environment. It covers:

- how screen configurations are modelled,
- how window positions and sizes are stored,
- how layouts are reconciled when screens change,
- how we ensure windows (especially plugin UIs) remain reachable and usable.

The goals are:

- predictable recall of window positions across sessions,
- robust behaviour when displays are added or removed,
- avoidance of off-screen or unresizable plugin windows,
- a unified model for native plugin windows and Aura-managed windows.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Screen Sets](#2-screen-sets)  
  - [2.1 Screen Descriptor](#21-screen-descriptor)  
  - [2.2 ScreenSet Identity](#22-screenset-identity)  
- [3. Window Descriptors](#3-window-descriptors)  
  - [3.1 Logical Window Keys](#31-logical-window-keys)  
  - [3.2 Geometry and Normalisation](#32-geometry-and-normalisation)  
  - [3.3 Modes](#33-modes)  
- [4. Layout Profiles](#4-layout-profiles)  
  - [4.1 Per-ScreenSet Layouts](#41-per-screenset-layouts)  
  - [4.2 Selecting a Layout Profile](#42-selecting-a-layout-profile)  
- [5. Screen Changes and Reconciliation](#5-screen-changes-and-reconciliation)  
  - [5.1 Matching ScreenSets](#51-matching-screensets)  
  - [5.2 Mapping Windows to the Current ScreenSet](#52-mapping-windows-to-the-current-screenset)  
- [6. Off-Screen Protection and Clamping](#6-off-screen-protection-and-clamping)  
- [7. Plugin UI Windows](#7-plugin-ui-windows)  
  - [7.1 Responsibility Split (Aura / Pulse / Signal)](#71-responsibility-split-aura--pulse--signal)  
  - [7.2 IPC Considerations](#72-ipc-considerations)  
- [8. Persistence](#8-persistence)  
- [9. Relationship to Other Documents](#9-relationship-to-other-documents)

---

## 1. Overview

Loophole must handle multi-screen setups gracefully. Typical scenarios include:

- docking the console or mixer full-screen on a secondary display,
- placing plugin UIs on a side monitor,
- closing and reopening windows with positions preserved,
- changing between “laptop only” and “laptop + external monitor” setups.

The system should:

- recall window positions **per screen configuration**, and
- automatically adjust positions when screens are added/removed or their
  resolution changes,
- guarantee that windows are always reachable and resizable.

This architecture applies to:

- native plugin UI windows created by Signal,
- Aura’s own windows and panels,
- Electron BrowserWindows used by the UI layer.

---

## 2. Screen Sets

### 2.1 Screen Descriptor

Aura queries the operating system for the current display configuration and
constructs a **Screen Descriptor** for each display, including:

- a stable OS display identifier (where possible),
- pixel resolution,
- scale factor (for HiDPI),
- origin (x, y) in the virtual desktop coordinate space,
- work area (excluding system UI such as menu bars and docks),
- primary/secondary flag.

These descriptors are used to track changes to the display topology.

### 2.2 ScreenSet Identity

A **ScreenSet** is the set of Screen Descriptors for all currently attached
displays.

Aura computes a **ScreenSet ID** by hashing a normalised representation of:

- the number of displays,
- each display’s stable identifier,
- relative arrangement (origin),
- key attributes such as resolution and scale.

This ScreenSet ID is used to:

- select an appropriate layout profile when the app starts,
- detect when the display configuration has changed and a different layout
  should be applied.

In addition to the ScreenSet ID, Aura uses a **machine identifier** (for
example, a hashed or otherwise stable identifier such as MAC address or other
OS-provided machine ID) to differentiate layout profiles per machine. This
machine identifier is used only for layout profile differentiation, not as a
security or DRM mechanism. It allows the same project to maintain distinct
window arrangements on different machines (e.g. a laptop vs a studio desktop)
while sharing the same ScreenSet configuration.

---

## 3. Window Descriptors

Windows (console, mixer, plugin UIs, etc.) are described by **Window
Descriptors**.

### 3.1 Logical Window Keys

Each window has a stable logical key, for example:

- `console-main`,
- `mixer-main`,
- `transport-bar`,
- `plugin-[trackId]-[nodeId]`,
- `meter-bridge`,
- `inspector-panel`.

This key is used to identify the window in layout storage, regardless of how
it is implemented (Electron, native, plugin, etc.).

### 3.2 Geometry and Normalisation

Window geometry is stored as:

- **absolute rectangle** in virtual desktop coordinates:

  - `x`, `y`, `width`, `height`,

- and/or **normalised rectangle** relative to a specific display’s work area:

  - `normX`, `normY`, `normWidth`, `normHeight` in the range 0–1.

The descriptor also stores the **target display identifier**, which may be:

- a specific screen ID (for windows anchored to a particular display),
- or a logical directive such as “primary display” or “same as console”.

When screen configurations change:

- if the target display still exists, the normalised geometry is projected
  into the new work area size,
- otherwise, a fallback display is chosen and the normalised geometry is
  applied there.

### 3.3 Modes

Each window has a **mode**, such as:

- `docked`,
- `floating`,
- `maximised`,
- `fullscreen`,
- `hidden`.

The combination of mode and geometry determines how Aura should restore the
window:

- `fullscreen` → occupy the full work area of the target display,
- `maximised` → occupy the work area minus any reserved UI regions,
- `floating` → use the stored rectangle,
- `docked` → managed by Aura’s layout/docking system rather than explicit
  rectangles.

---

## 4. Layout Profiles

### 4.1 Per-ScreenSet Layouts

Aura maintains **Layout Profiles** that can be stored both per-project and in
user/global configuration.

For **project-level layout storage**, profiles are keyed by both the current
machine identifier and ScreenSet ID: `(machineId, ScreenSetId)`. This allows:

- the same project opened on different machines to have distinct window
  arrangements,
- the same machine to maintain multiple layouts for different ScreenSets
  (e.g. laptop-only vs laptop + external display).

A project can store multiple layout profiles—one per `(machineId, ScreenSetId)`
combination that has been used with that project.

For **global/user-level layouts**, Aura may maintain layout defaults outside
the project, keyed by ScreenSet ID, used for new projects or when
project-specific data is missing.

Each Layout Profile contains:

- a set of Window Descriptors (by window key),
- any additional docking-specific layout information.

### 4.2 Selecting a Layout Profile

On startup or whenever the display configuration changes:

1. Aura computes the current ScreenSet ID and machine identifier.
2. For project-level layouts:
   - Aura first tries to load a layout profile matching the current
     `(machineId, ScreenSetId)`.
   - If none is found, it may try another profile for the same `machineId` with
     a "closest" ScreenSet, or proceed to global/user defaults.
3. For global/user-level layouts:
   - If no project-specific profile is available, Aura checks for a global
     layout profile for the current ScreenSet ID.
4. If still no match, Aura may:

   - find the "closest" known ScreenSet (e.g. shares most display IDs), or
   - fall back to a factory-default layout and create a new profile for this
     `(machineId, ScreenSetId)` combination.

---

## 5. Screen Changes and Reconciliation

### 5.1 Matching ScreenSets

When changing from one ScreenSet to another (e.g. removing an external
monitor):

- Aura identifies:

  - which displays are unchanged (same ID),
  - which have been removed,
  - which are new.

This allows per-window decisions about which layout data can be reused.

### 5.2 Mapping Windows to the Current ScreenSet

For each Window Descriptor:

1. If the target display ID still exists:
   - project the window’s normalised rectangle onto that display’s work area.
2. If the target display ID does not exist:
   - choose a fallback display (typically the primary display, or a user
     preference),
   - project the normalised rectangle onto the fallback display’s work area.
3. Apply clamping rules (see below) to ensure the window remains visible and
   usable.

---

## 6. Off-Screen Protection and Clamping

To avoid off-screen or unresizable windows (particularly plugin UIs), Aura
applies **clamping rules** before showing any window:

- Compute the union of all display work areas.
- If a window’s rectangle lies completely outside this union:

  - reposition it so that at least the title bar or top-left corner lies
    within a visible work area.

- If a window’s size exceeds the chosen display’s work area:

  - clamp its width and height to fit within the work area,
  - ensure at least a minimal visible area (e.g. a region sufficient to grab
    and resize).

Even if a plugin requests a size that would be larger than the current
display, Aura and Signal cooperate to constrain the actual window geometry so
that:

- the user can always access drag handles or resize corners,
- no case exists where all corners are off-screen.

---

## 7. Plugin UI Windows

### 7.1 Responsibility Split (Aura / Pulse / Signal)

- **Signal** is responsible for:

  - creating and owning native plugin UI windows,
  - attaching them to the appropriate OS parent (for focus, z-order),
  - respecting size constraints when requested by Aura.

- **Aura** is responsible for:

  - deciding where plugin windows should appear (target display, rectangle),
  - managing layout profiles per ScreenSet,
  - ensuring plugin windows remain reachable and visible.

- **Pulse** is responsible for:

  - coordinating plugin UI lifecycle and routing between Aura and Signal,
  - storing minimal plugin UI state in the project where appropriate
    (e.g. “this plugin has a UI open” flag, if needed),
  - carrying position/size commands between Aura and Signal.

### 7.2 IPC Considerations

The Plugin UI IPC domain (specified separately) will include commands such as:

- `pluginUi.openWindow`  
  Request opening a plugin UI for a specific Node.

- `pluginUi.closeWindow`  
  Request closing the plugin UI.

- `pluginUi.setWindowLayout`  
  Request placing the plugin window at a specific rectangle in virtual desktop
  coordinates, corresponding to the Window Descriptor computed by Aura.

- `pluginUi.windowGeometryChanged`  
  Event from Signal to Pulse/Aura when a plugin window is resized by the user
  or by the plugin itself (e.g. when the plugin changes its own UI size).

Aura then:

- updates the Window Descriptor for that plugin window,
- writes updated layout data back into the current Layout Profile.

### Clip-Scoped Plugin Editors (ARA & Future)

Certain plugin UIs operate in clip context rather than track context.
When Aura opens an ARA-capable plugin as a clip editor, the UI receives:

- clipId

- laneId

- regionId or groupId

- bounds of the edited section

This enables plugin UIs to present a clip-level editing surface (e.g.
Melodyne-style), rather than the traditional track-insert view.

Window placement and persistence follow the same rules as normal plugin
windows, but clip editors may also dock into dedicated editing panels
within Aura.

---

## 8. Persistence

Window layout profiles are persisted at two levels:

### Project-Level Persistence

Each project stores window/layout information keyed by `(machineId, ScreenSetId)`
in project metadata. This allows:

- the same project to remember different layouts on a laptop, a studio desktop,
  etc.,
- each machine to maintain distinct layouts for different ScreenSet
  configurations used with that project.

When opening a project, Aura first tries to load a layout profile matching the
current `(machineId, ScreenSetId)`. If none is found, it falls back to:

- another profile for the same `machineId` with a "closest" ScreenSet, or
- a global/user default for the current ScreenSet, or
- a factory-default layout.

### User/Global-Level Persistence

Aura maintains generic layout defaults for each ScreenSet in user/global
configuration (outside project files). These defaults are used:

- for new projects,
- when project-specific data is missing or unavailable.

Project-specific layout profiles take precedence over global defaults when
available.

Plugin UI "open/closed" state might be treated either as:

- session-only, or
- project-level preference, depending on design decisions made in the Plugin
  UI IPC spec.

In all cases, persistent storage must respect ScreenSet IDs (and, for
project-level storage, machine identifiers) to avoid applying layouts that
belong to incompatible display configurations or different machines.

---

## 9. Relationship to Other Documents

This architecture interacts with:

- [Aura Architecture](./A04-aura.md) – Aura responsibilities and UI scope,
- [Signal Architecture](./A03-signal.md) – Signal's management of plugin
  instances and native UI,
- [Tracks, Lanes & Roles](./B01-tracks-lanes-and-roles.md) – identity
  of Nodes and context for plugin UIs,
- Plugin UI IPC spec (`docs/specs/ipc/pulse/plugin-ui.md`) – command and
  event definitions for managing plugin window lifecycle and geometry.

It also complements more general UI layout and docking architecture, which
may be documented separately within the Aura-specific design documents.
