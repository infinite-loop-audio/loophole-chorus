# Signal IPC – Plugin UI Domain

This document defines the **Plugin UI** IPC domain between Pulse and Signal.

The Plugin UI domain is responsible for:

- creating and managing **native plugin UI windows** for plugin instances,
- embedding plugin UIs into Aura-hosted surfaces where supported,
- applying and enforcing window geometry (position, size, visibility),
- handling plugin-driven resizes and focus requests,
- ensuring plugin windows remain **recoverable and visible** across changing
  screen configurations.

Pulse owns:

- which plugin UIs should be open,
- how plugin UIs are arranged in the overall Aura layout,
- persistence of window layouts per machine + screen set,
- mapping plugin UI windows to project-level concepts (channels, tracks, etc.).

Signal owns:

- actual interaction with plugin UI APIs (VST3/CLAP/AU/etc.),
- creating native windows or attaching plugin views to host surfaces,
- enforcing window constraints, safe positioning, and DPI awareness,
- relaying plugin-driven changes back to Pulse.

Aura never talks directly to Signal. All control flows **Pulse → Signal**, with
Pulse maintaining a separate IPC channel with Aura to synchronise window state.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Plugin UI domain:

- opens and closes plugin UIs for specific `pluginInstanceId`s,
- attaches plugin UIs to:
  - **external** native windows managed by Signal, or
  - **embedded** host surfaces created by Aura (via platform-specific handles),
- applies window rects (position and size) provided by Pulse,
- reports plugin-driven:
  - resizes,
  - visibility changes,
  - focus requests,
- ensures windows remain **on-screen and resizable** when screen sets change.

### 1.2 Non-Goals

The Plugin UI domain does **not**:

- change plugin parameters (handled by parameter/automation/gesture domains),
- decide when to show/hide plugin views (that’s Pulse/Aura policy),
- store or reason about long-term layout preferences (Pulse does this, with
  Composer as an optional metadata layer),
- manage non-plugin windows (mixers, editors, browsers): those are Aura-only.

---

## 2. Identity & Modes

### 2.1 Plugin Instance

Plugin UIs are always attached to a specific **plugin instance**:

```json
{
  "pluginInstanceId": "pluginInstance:abc123"
}
```

This comes from the `signal.plugin` domain.

### 2.2 UI Window ID

Pulse assigns a **UI Window ID** for each plugin UI it wants to manage:

```json
{
  "uiWindowId": "ui:plugin:track:42:slot:1"
}
```

- `uiWindowId` is stable within the Pulse/Aura layout model.
- Multiple `uiWindowId`s may refer to the same `pluginInstanceId` in unusual
  cases (e.g. multi-view UIs), though normally there’s a 1:1 mapping.

### 2.3 UI Modes

Two main integration modes:

- `"external"` – Signal creates a native top-level window:
  - OS-managed window (HWND, NSWindow, X11/Wayland window, etc.),
  - Aura treats it as an external window and tracks it via events.
- `"embedded"` – Aura creates a host surface and passes a platform-specific
  **host handle** to Signal:
  - Signal attaches the plugin’s UI as a child view/window of that host,
  - used when embedding plugin UIs inside Electron panels.

---

## 3. Host Window Handles

For `"embedded"` mode, Pulse provides:

```json
{
  "hostWindow": {
    "platform": "macOS",   // "macOS" | "Windows" | "X11" | "Wayland" | etc.
    "kind": "view",        // "view" | "window" | other platform-specific flavours
    "handle": "opaque-string-or-token"
  }
}
```

- `handle` is an opaque token understood by Signal’s platform layer.
- Pulse receives it from Aura; Signal **never** talks directly to Aura.

For `"external"` mode, no host handle is required; Signal manages the top-level
window itself.

---

## 4. Commands (Pulse → Signal)

### 4.1 `pluginUI.open`

Open a plugin UI for a given plugin instance.

**Request**

```json
{
  "command": "pluginUI.open",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "pluginInstanceId": "pluginInstance:abc123",
  "mode": "external", // "external" | "embedded"
  "initialRect": {
    "x": 1200,
    "y": 100,
    "width": 800,
    "height": 600
  },
  "hostWindow": null,
  "options": {
    "visible": true,
    "alwaysOnTop": false,
    "allowResize": true
  }
}
```

For `"embedded"` mode, `hostWindow` must be provided, e.g.:

```json
"mode": "embedded",
"hostWindow": {
  "platform": "Windows",
  "kind": "hwnd",
  "handle": "0x00123456"
}
```

**Response**

```json
{
  "replyTo": "pluginUI.open",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "pluginInstanceId": "pluginInstance:abc123",
  "mode": "external",
  "actualRect": {
    "x": 1180,
    "y": 90,
    "width": 820,
    "height": 620
  },
  "constraints": {
    "minWidth": 400,
    "minHeight": 300,
    "maxWidth": null,
    "maxHeight": null,
    "resizable": true
  }
}
```

Semantics:

- Signal:
  - creates/attaches the plugin UI,
  - clamps the requested rect to **visible, safe** screen bounds,
  - applies plugin’s size constraints (if the plugin exposes them),
  - returns the resulting rect and constraints.
- Pulse:
  - updates its layout model with `actualRect` and `constraints`,
  - uses events to keep Aura in sync.

Signal may also emit `pluginUI.opened` event (see below).

---

### 4.2 `pluginUI.close`

Close a plugin UI window.

**Request**

```json
{
  "command": "pluginUI.close",
  "uiWindowId": "ui:plugin:track:42:slot:1"
}
```

Semantics:

- If the plugin UI is open, Signal:
  - closes/detaches the native UI,
  - releases any associated resources for this UI window.
- The plugin instance itself is **not** destroyed; only its UI.

Signal should emit `pluginUI.closed` with `reason: "normal"`.

---

### 4.3 `pluginUI.setRect`

Request a move/resize of the plugin UI window.

**Request**

```json
{
  "command": "pluginUI.setRect",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "rect": {
    "x": 1300,
    "y": 120,
    "width": 900,
    "height": 650
  }
}
```

Semantics:

- Coordinates are in the same **virtual desktop** coordinate system that Aura
  uses and translates to on each platform.
- Signal:
  - applies the rect to the native window/view,
  - clamps to visible, on-screen bounds,
  - respects min/max size constraints,
  - then emits `pluginUI.rectChanged` with the resulting `actualRect`.

This guarantees that plugin windows remain recoverable even if a stored layout
would place them off-screen.

---

### 4.4 `pluginUI.setVisibility`

Show or hide the plugin UI.

**Request**

```json
{
  "command": "pluginUI.setVisibility",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "visible": true
}
```

Semantics:

- For `"external"` windows, Signal:
  - shows/hides the OS window.
- For `"embedded"` mode, Signal:
  - shows/hides the plugin child view within the host window.

Signal emits `pluginUI.visibilityChanged`.

---

### 4.5 `pluginUI.setZOrderHints`

Set Z-order hints for a plugin UI window.

**Request**

```json
{
  "command": "pluginUI.setZOrderHints",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "options": {
    "alwaysOnTop": false
  }
}
```

Semantics:

- Signal applies hints as far as the OS and windowing model allow.
- Aura still controls overall layout ordering within its own UI; this only
  affects native top-level behaviour where relevant.

---

### 4.6 `pluginUI.updateHostWindow`

Update the host window/surface for an embedded plugin UI.

**Request**

```json
{
  "command": "pluginUI.updateHostWindow",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "hostWindow": {
    "platform": "macOS",
    "kind": "view",
    "handle": "opaque-updated-token"
  }
}
```

Semantics:

- Used when:
  - Aura re-creates or moves the host surface (e.g. panel docking changes).
- Signal:
  - re-attaches the plugin UI to the new host,
  - minimises flicker and reallocation.

---

### 4.7 `pluginUI.requestFocus`

Request that the plugin window gains keyboard/mouse focus.

**Request**

```json
{
  "command": "pluginUI.requestFocus",
  "uiWindowId": "ui:plugin:track:42:slot:1"
}
```

Semantics:

- Signal attempts to give focus to the plugin UI.
- On some platforms, OS rules may prevent this (e.g. focus-stealing policies);
  Signal should do best-effort and may emit a warning if it fails.

---

## 5. Events (Signal → Pulse)

### 5.1 `pluginUI.opened`

Emitted once a plugin UI has been successfully opened and attached.

**Payload**

```json
{
  "event": "pluginUI.opened",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "pluginInstanceId": "pluginInstance:abc123",
  "mode": "external",
  "rect": {
    "x": 1180,
    "y": 90,
    "width": 820,
    "height": 620
  },
  "constraints": {
    "minWidth": 400,
    "minHeight": 300,
    "maxWidth": null,
    "maxHeight": null,
    "resizable": true
  }
}
```

Pulse forwards this information to Aura to update its layout model.

---

### 5.2 `pluginUI.closed`

Emitted when a plugin UI window is closed.

**Payload**

```json
{
  "event": "pluginUI.closed",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "pluginInstanceId": "pluginInstance:abc123",
  "reason": "normal" // "normal" | "pluginClosed" | "error" | "crash"
}
```

Semantics:

- `pluginClosed` – plugin itself requested closing its UI.
- `error` – some recoverable error caused the window to close.
- `crash` – plugin crash or serious error tied to this UI.

Pulse must:

- keep its layout model consistent,
- potentially re-open the UI if the user requests it again.

---

### 5.3 `pluginUI.rectChanged`

Emitted when the plugin UI window’s rect changes:

- plugin-driven resize,
- OS-driven reposition (e.g. due to screen reconfiguration),
- clamping of a requested rect.

**Payload**

```json
{
  "event": "pluginUI.rectChanged",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "rect": {
    "x": 1200,
    "y": 110,
    "width": 840,
    "height": 640
  }
}
```

Pulse:

- updates stored layout,
- notifies Aura to reflect the new position/size in its internal layout model.

---

### 5.4 `pluginUI.visibilityChanged`

Emitted when the plugin UI becomes visible/hidden.

**Payload**

```json
{
  "event": "pluginUI.visibilityChanged",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "visible": true
}
```

Causes may include:

- user closing/minimising the window via OS controls,
- plugin hiding its own UI.

---

### 5.5 `pluginUI.focusRequested`

Plugin requests focus for its UI (e.g. user clicked inside an embedded region).

**Payload**

```json
{
  "event": "pluginUI.focusRequested",
  "uiWindowId": "ui:plugin:track:42:slot:1"
}
```

Semantics:

- Pulse forwards this to Aura.
- Aura decides how to honour the request, following platform and UI rules.

---

### 5.6 `pluginUI.error`

Error relating to plugin UI creation or management.

**Payload**

```json
{
  "event": "pluginUI.error",
  "uiWindowId": "ui:plugin:track:42:slot:1",
  "pluginInstanceId": "pluginInstance:abc123",
  "code": "createFailed", // "createFailed" | "attachFailed" | "unsupported" | "other"
  "message": "Plugin does not provide a native UI.",
  "data": {}
}
```

Pulse should:

- show appropriate feedback,
- potentially fall back to a generic parameter editor,
- avoid repeatedly trying to open a UI that is not supported.

---

## 6. Safe Positioning & Screen Changes

Signal must ensure that plugin UI windows remain **recoverable**:

- If a requested `rect` would place a window completely off-screen:
  - clamp it to visible bounds,
  - emit `pluginUI.rectChanged` with the adjusted rect.
- If OS screen configuration changes (e.g. monitor removed, resolution change):
  - OS will typically adjust windows; Signal should:
    - detect meaningful changes when possible,
    - emit `pluginUI.rectChanged` events accordingly.

Pulse is responsible for:

- associating window rects with:
  - `machineId`,
  - `screenSetId` (from the window management/hardware model),
- applying different saved layouts when screen sets change,
- orchestrating layout recovery (e.g. repositioning many windows at once).

---

## 7. Error Handling (Commands)

Commands can fail synchronously for reasons including:

- unknown `pluginInstanceId` or `uiWindowId`,
- plugin has no UI,
- platform does not support requested `mode`,
- invalid or unusable `hostWindow` handle.

Example error:

```json
{
  "error": {
    "code": "noPluginUI",
    "message": "The plugin instance does not provide a native UI.",
    "domain": "pluginUI",
    "data": {
      "pluginInstanceId": "pluginInstance:abc123"
    }
  }
}
```

Pulse must treat these as authoritative and:

- avoid retrying the same invalid requests,
- adjust UI options (e.g. hide “Open UI” button for this plugin instance).

---

## 8. Relationship to Other Signal Domains

- **signal.plugin**  
  Provides `pluginInstanceId` and ensures instances exist before UIs are opened.

- **signal.hardware / window management architecture**  
  Screen set and machine identification inform layout logic; while Signal does
  not own those IDs, it must respect and adapt to the effective screen layout.

- **signal.diagnostics**  
  Very heavy or misbehaving plugin UIs may correlate with performance issues;
  diagnostics can expose patterns (though Plugin UI itself is mostly GUI-side).

- **Pulse Plugin UI domain**  
  Pulse’s plugin UI spec:
  - defines how Aura asks for plugin UIs to be opened,
  - stores all layout metadata per project/machine/screen-set,
  - mediates between Aura’s view of layout and Signal’s native window reality.

---

## 9. Extensibility

Future extensions may include:

- per-plugin UI theme hints (dark/light, scaling),
- multi-view plugins with several UIs per instance,
- tabbed/plugin containers managed by Aura but backed by Signal child views,
- remote plugin UIs (e.g. network-hosted plugins, remote desktop scenarios),
- capturing plugin UI snapshots for thumbnails and Composer metadata.

Any extensions should respect the core design:

- Pulse controls **when and where** plugin UIs exist,
- Signal controls **how** they are realised natively,
- Aura presents the final UI layout to the user.
