# Aura Architecture

Aura is the user interface layer of Loophole. It is responsible for all visual
representation, user interaction, editing workflows, and creation of high-level
intents that Pulse resolves into authoritative model changes. Aura does not
store project state. It mirrors Pulse, sends editing requests, and renders the
resulting model.

Aura must be expressive, performant and capable of representing extremely rich
editorial structures, while maintaining a strict separation from engine and
data-layer concerns.

---

## Contents

- [1. Overview](#1-overview)
- [2. Responsibilities](#2-responsibilities)
- [3. Architectural Position](#3-architectural-position)
- [4. State Model](#4-state-model)
  - [4.1 No Authoritative State](#41-no-authoritative-state)
  - [4.2 Local Ephemeral State](#42-local-ephemeral-state)
- [5. Interaction with Pulse](#5-interaction-with-pulse)
  - [5.1 Editing Intents](#51-editing-intents)
  - [5.2 Snapshot Consumption](#52-snapshot-consumption)
  - [5.3 Error Handling](#53-error-handling)
- [6. Interaction with Signal](#6-interaction-with-signal)
  - [6.1 Plugin Windows](#61-plugin-windows)
  - [6.2 Transport Controls](#62-transport-controls)
  - [6.3 Metering and Analysis](#63-metering-and-analysis)
- [7. Rendering Model](#7-rendering-model)
  - [7.1 Tracks and Timeline](#71-tracks-and-timeline)
  - [7.2 Clips and Lanes](#72-clips-and-lanes)
  - [7.3 Mixer](#73-mixer)
  - [7.4 Parameter and Automation Views](#74-parameter-and-automation-views)
  - [7.5 Plugin Browser](#75-plugin-browser)
- [8. Interaction with Composer](#8-interaction-with-composer)
- [9. Control Surfaces and Hardware](#9-control-surfaces-and-hardware)
  - [9.1 Context Signals](#91-context-signals)
  - [9.2 Hardware Management UI](#92-hardware-management-ui)
  - [9.3 Learn Mode](#93-learn-mode)
- [10. UI/UX Considerations](#10-uiux-considerations)
  - [10.1 Deterministic Rendering](#101-deterministic-rendering)
  - [10.2 Visual Virtualisation](#102-visual-virtualisation)
  - [10.3 High-Frequency Updates](#103-high-frequency-updates)
- [11. Extensibility](#11-extensibility)
- [12. Future Extensions](#12-future-extensions)

---

## 1. Overview

Aura is the front-end interface for Loophole, built using web technologies
(Electron + TypeScript). It provides:

- timeline editors,
- mixers,
- media browsers,
- parameter inspectors,
- automation curves,
- plugin UIs,
- routing and metadata panels.

Aura acts as the **view** and **controller** in a strict MVC relationship:

- **Pulse**: Model
- **Aura**: View + controller (intents only)
- **Signal**: Executor (real-time engine)

Aura issues intents; Pulse validates and updates the model; Aura re-renders the
new state.

---

## 2. Responsibilities

Aura is responsible for:

1. **Rendering**
   Visualising Tracks, Clips, Lanes, Channels, nodes, automation and
   parameters.

2. **User Interaction**
   All mouse, keyboard, touch and gesture-based editing.

3. **Intent Generation**
   Converting user interactions into editing commands for Pulse.

4. **Plugin UI Embedding**
   Hosting plugin windows provided by Signal.

5. **Feedback Presentation**
   Transport state, metering, analysis, warnings, errors.

Aura does **not** interpret routing, construct audio graphs, or manage real-time
state.

---

## 3. Architectural Position

Aura sits at the top of the Loophole stack:

```
[ Aura ]         – UI, editing, user intent
    ↓
[ Pulse ]        – model, structure, routing, parameters
    ↓
[ Signal ]       – real-time DSP and plugins
```

Aura communicates:

- **upward** to the user,
- **downward** by sending intents to Pulse,
- **laterally** by displaying plugin UIs hosted by Signal.

Aura never bypasses Pulse for structural or project data.

---

## 4. State Model

### 4.1 No Authoritative State

Aura stores no project data. It maintains only:

- local UI state (zoom, selections, highlighting),
- pending gestures,
- plugin window embeddings.

All project-critical data lives in Pulse.

### 4.2 Local Ephemeral State

Examples of ephemeral UI state:

- currently edited automation curve,
- selection rectangles,
- drag previews,
- ghost clips during move operations,
- piano roll navigation state.

Ephemeral states can be discarded at any time; Pulse remains authoritative.

---

## 5. Interaction with Pulse

### 5.1 Editing Intents

Aura sends high-level intents such as:

- “split clip at time X”,
- “create audio lane”,
- “move clip to track N”,
- "add instrument node",
- "set parameter X to value Y",
- "route lane to node Z".

Pulse validates and executes (or rejects) these intents.

### 5.2 Snapshot Consumption

Pulse emits snapshot events whenever the model changes. A snapshot contains:

- the subset of the project relevant to the UI,
- updated sections only,
- identifiers for changed entities,
- structural diffs where appropriate.

Aura diff-applies snapshots and re-renders relevant components.

### 5.3 Error Handling

When Pulse rejects an intent:

- Aura displays contextual errors,
- highlights offending UI elements,
- offers corrective suggestions,
- may revert local UI previews.

Pulse never sends errors directly to Signal.

---

## 6. Interaction with Signal

### 6.1 Plugin Windows

Signal hosts plugin UIs as native windows and provides OS-level handles to Aura.
Aura embeds these windows via platform-specific APIs.

Aura must:

- manage positions,
- remember visibility,
- track focus states.

### 6.2 Transport Controls

Aura sends transport commands to Pulse (play, pause, seek).
Signal receives transport state only from Pulse, never directly from Aura.

### 6.3 Metering and Analysis

Metering and analysis data (levels, spectrograms, waveforms) arrives via a
high-rate side channel from Signal. Aura visualises these without affecting
routing or structure.

---

## 7. Rendering Model

### 7.1 Tracks and Timeline

Aura renders:

- nested Tracks,
- timeline grids,
- Clip boundaries,
- lane summary info,
- automation lanes,
- playback cursor.

Track visualisation updates from Pulse snapshot deltas.

### 7.2 Clips and Lanes

Aura displays Clips, each showing:

- its duration,
- contents of its Lanes,
- per-lane editors (audio, MIDI, automation, video).

Because Clips define Lanes, Aura dynamically allocates editors based on Clip content.

### 7.3 Mixer

Aura renders:

- Channels (when attached to Tracks),
- nodes in order,
- LaneStreamNode(s) (often shortened to 'LaneStream'),
- inserts, sends (future),
- meters and gain/pan.

Mixer structure is defined entirely by Pulse.

### 7.4 Parameter and Automation Views

Aura shows:

- parameter inspectors,
- logical parameter groupings,
- automation lanes,
- modulation sources/destinations (future).

Parameter semantics may be enhanced by Composer but are always resolved through Pulse.

### 7.5 Plugin Browser

Aura shows plugin lists based on:

- categories,
- tags,
- metadata (possibly Composer-enhanced),
- user history.

Pulse provides the available node list; Aura never queries Signal.

---

## 8. Interaction with Composer

Aura **never communicates with Composer**.

Composer metadata reaches Aura only through Pulse, embedded in:

- parameter groupings,
- semantic roles,
- plugin categorisation,
- mapping suggestions.

Aura is a consumer of metadata, not a participant in metadata inference.

---

## 9. Control Surfaces and Hardware

Aura participates in the control-surfaces ecosystem by providing UI for hardware management and visualising mapping state.

### 9.1 Context Signals

Aura emits **context signals** to Pulse indicating:

- which view is active (Arranger, Mixer, Launcher, Plugin UI, Clip Editor),
- which track/clip/plugin is focused,
- which mixer bank is in view.

These context signals enable Pulse to switch mapping sets automatically, ensuring hardware controls adapt to the current view.

### 9.2 Hardware Management UI

Aura provides UIs for:

- hardware overview and status (connected devices, capabilities),
- "Learn" mode and mapping editing (creating and modifying mapping rules),
- per-device mapping visualisation (showing what each control does),
- feedback/diagnostic overlays ("Controller X mapped to Y").

Aura does **not** perform mapping logic; it:
- reflects Pulse's mapping decisions in the UI,
- passes user edits (e.g. new mapping, delete mapping) back to Pulse as editing intents.

### 9.3 Learn Mode

When the user activates "Learn" mode:

1. Aura enters a special UI state highlighting the target parameter/action,
2. User moves a hardware control,
3. Signal forwards the control event to Pulse,
4. Pulse detects the active parameter/target in UI context and creates a MappingRule override,
5. Aura updates the UI to show the new mapping.

All mapping decisions remain in Pulse; Aura only provides the UI workflow.

---

## 10. UI/UX Considerations

### 10.1 Deterministic Rendering

Aura should render deterministically based on Pulse snapshots:

- no hidden local state,
- derived UI always consistent with the model.

### 10.2 Visual Virtualisation

Aura must virtualise:

- track lists,
- clip regions,
- MIDI notes,
- automation points.

This is essential for performance in large projects.

### 10.3 High-Frequency Updates

Aura may receive large volumes of metering/analysis data.
It must:

- throttle rendering,
- batch updates,
- schedule painting efficiently.

These updates do not affect project state.

---

## 11. Extensibility

The architecture is designed for extensions such as:

- custom editors for new Lane types,
- additional mixer panels,
- floating tool windows,
- modular inspectors,
- scripting overlays (future).

Pulse remains the authority; Aura only displays new capabilities.

---

## 12. Future Extensions

- collaborative editing (multiple Aura clients),
- spatial audio editors,
- microtiming/grids per Clip,
- advanced video editing panels,
- device control dashboards,
- interactive harmonic tools.

These will extend Aura without impacting Pulse or Signal responsibilities.
