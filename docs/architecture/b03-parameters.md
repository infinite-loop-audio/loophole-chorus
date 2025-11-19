# Parameters Architecture

This document defines how parameters are represented, identified, mapped,
automated and resolved across the Loophole system. Parameters are central to
automation, modulation, mixing, node control and UI interactions. The model
must support stability across plugin formats and versions, and provide resilience
in the face of plugin updates or replacements.

This is an architectural document. It covers conceptual responsibilities and
behaviours, not IPC or runtime specifics.

---

## Contents

- [1. Overview](#1-overview)
- [2. Parameter Identity](#2-parameter-identity)
  - [2.1 Fully Qualified Parameter IDs](#21-fully-qualified-parameter-ids)
  - [2.2 Logical vs Variant Parameters](#22-logical-vs-variant-parameters)
  - [2.3 Stability Requirements](#23-stability-requirements)
- [3. Parameter Types](#3-parameter-types)
  - [3.1 Continuous Parameters](#31-continuous-parameters)
  - [3.2 Discrete Parameters](#32-discrete-parameters)
  - [3.3 Compound Parameters](#33-compound-parameters)
  - [3.4 Hidden/Internal Parameters](#34-hiddeninternal-parameters)
- [4. Parameter Aliasing](#4-parameter-aliasing)
  - [4.1 Purpose](#41-purpose)
  - [4.2 Local Alias Table](#42-local-alias-table)
  - [4.3 Alias Resolution Lifecycle](#43-alias-resolution-lifecycle)
- [5. Semantic Roles](#5-semantic-roles)
  - [5.1 Purpose of Semantic Roles](#51-purpose-of-semantic-roles)
  - [5.2 Relationship to Plugin Parameters](#52-relationship-to-plugin-parameters)
- [6. Automation](#6-automation)
  - [6.1 Parameter Binding](#61-parameter-binding)
  - [6.2 Data Storage](#62-data-storage)
  - [6.3 Gesture vs Playback Streams](#63-gesture-vs-playback-streams)
- [7. Parameter Mapping and Remapping](#7-parameter-mapping-and-remapping)
  - [7.1 Plugin Version Changes](#71-plugin-version-changes)
  - [7.2 Format Changes](#72-format-changes)
  - [7.3 Plugin Replacement](#73-plugin-replacement)
- [8. Interaction with Composer](#8-interaction-with-composer)
  - [8.1 Composer’s Role](#81-composers-role)
  - [8.2 Metadata Used by Pulse](#82-metadata-used-by-pulse)
  - [8.3 Optional and Advisory Nature](#83-optional-and-advisory-nature)
- [9. Signal Behaviour](#9-signal-behaviour)
- [10. Future Extensions](#10-future-extensions)

---

## 1. Overview

Parameters represent any adjustable value within Loophole:

- plugin parameters,
- Track/channel parameters (gain, pan, sends),
- device controls,
- global parameters (tempo, swing),
- Clip-specific expressions (future).

Pulse is responsible for maintaining authoritative parameter state. Signal
receives resolved, real-time-safe parameter updates. Aura interacts with Pulse
to display and edit parameters but holds no authoritative state.

---

## 2. Parameter Identity

Parameter identity must be stable, unambiguous and independent of UI layout,
plugin format or vendor naming.

### 2.1 Fully Qualified Parameter IDs

All parameters use **fully qualified IDs** constructed from:

```
entityType.entityId.nodeId.parameterId
```

Examples:

```
track.12.channel.gain
track.5.node.7.cutoff
clip.19.lane.3.automation
```

Plugin parameters use node-scoped parameter IDs.

Qualified IDs guarantee uniqueness even under:

- Track duplication,
- Clip duplication,
- multiple instances of the same plugin,
- nested Tracks.

### 2.2 Logical vs Variant Parameters

Pulse distinguishes:

- **Variant parameters** — exposed by a specific plugin version/format.
- **Logical parameters** — family-level stable identifiers representing “the same parameter” across versions/formats.

Variant parameters change over time; logical parameters remain stable.

### 2.3 Stability Requirements

Parameter IDs MUST remain stable:

- across saves,
- across undo/redo,
- across simple node insertions/removals,
- across plugin format switches,
- across plugin updates (where mapping is possible).

Pulse manages this stability using alias tables and, optionally, Composer metadata.

---

## 3. Parameter Types

### 3.1 Continuous Parameters
Floating-point values within a range, often with non-linear scaling:

- gain,
- frequency,
- resonance,
- mix,
- pan.

### 3.2 Discrete Parameters
Enumerations or stepped integer values:

- waveform type,
- filter mode,
- oversampling options.

### 3.3 Compound Parameters
Grouped parameters that conceptually belong together:

- ADSR envelopes,
- vector/pad controls,
- multi-axis parameters.

Compound parameters appear as separate automation targets but may share UI grouping.

### 3.4 Hidden/Internal Parameters
Plugins often expose parameters used for internal state but not intended for automation. Pulse may hide such parameters by default.

---

## 4. Parameter Aliasing

### 4.1 Purpose

Parameter aliasing preserves automation and state when plugin versions/formats
change. If a parameter moves or is renamed, Pulse resolves the old ID to a new one.

### 4.2 Local Alias Table

Pulse maintains a persistent alias table that maps:

```
oldParameterId → newParameterId
```

This table is:

- local to the project,
- saved with the project file,
- unaffected by Composer availability,
- used before Composer metadata.

### 4.3 Alias Resolution Lifecycle

When a plugin variant changes:

1. Pulse attempts direct matching (parameter name, index, metadata).
2. Pulse checks local alias table.
3. Pulse may request additional metadata from Composer (roles, logical IDs, stability hints).
4. Pulse finalises mapping and updates the alias table if needed.

Pulse, not Composer, is the authority.

---

## 5. Semantic Roles

### 5.1 Purpose of Semantic Roles

Semantic roles classify parameters by their meaning, not their location or name:

- `filter.cutoff`
- `filter.resonance`
- `osc.tune`
- `fx.mix`
- `amp.attack`

Roles are used for:

- automation remapping,
- plugin replacement,
- UI grouping and navigation,
- cross-plugin similarity features.

### 5.2 Relationship to Plugin Parameters

A parameter MAY have:

- no role,
- one primary role,
- multiple context-specific roles (rare).

Roles do not override parameter identity; they provide guidance for Pulse when
mapping or displaying parameters.

---

## 6. Automation

### 6.1 Parameter Binding

An automation Lane binds to a parameter via its fully qualified ID. Pulse
resolves the target parameter whenever:

- the plugin changes,
- the Clip moves,
- routing changes,
- automation is pasted to another Track.

### 6.2 Data Storage

Automation curves store:

- timestamp (timeline or Clip-local),
- value (normalised or absolute),
- interpolation mode.

Automation is stored at Clip scope but resolved at playback time.

### 6.3 Gesture vs Playback Streams

Pulse distinguishes:

- **gesture streams** — high-rate UI movements, passed to Signal via the high-rate channel,
- **playback streams** — automation evaluated during playback.

Signal treats all incoming automation uniformly, but origin paths differ.

---

## 7. Parameter Mapping and Remapping

### 7.1 Plugin Version Changes

When a plugin updates:

- Pulse compares variant parameter metadata,
- resolves moved/renamed parameters,
- uses alias tables and Composer metadata,
- preserves automation whenever possible.

### 7.2 Format Changes

When switching between VST3, CLAP or AU:

- Pulse identifies plugin family,
- locates logical parameter equivalents,
- applies format-specific scaling if required,
- confirms or updates automation targets.

### 7.3 Plugin Replacement

When the user replaces a node:

- Pulse uses semantic roles and Composer metadata to suggest likely equivalents,
- user can confirm or override mappings,
- automation and modulation MAY be preserved if roles align.

---

## 8. Interaction with Composer

### 8.1 Composer’s Role

Composer is an external metadata service that provides:

- normalised plugin identity,
- logical parameter identifiers,
- parameter metadata (names, ranges, scaling),
- semantic roles,
- mapping suggestions across versions, formats and plugin families.

Composer does **not** dictate behaviour. Pulse retains full authority.

### 8.2 Metadata Used by Pulse

Pulse MAY use Composer data to:

- interpret parameter semantics,
- improve aliasing suggestions,
- interpret ambiguous cases during plugin updates,
- suggest automation remapping,
- populate UI metadata for Aura.

### 8.3 Optional and Advisory Nature

Composer is optional.

If Composer is unavailable:

- Pulse falls back to local aliasing,
- parameter identity and automation remain functional,
- no loss of project integrity occurs.

Signal is never aware of Composer at any stage.

---

## 9. Signal Behaviour

Signal receives:

- resolved parameter IDs,
- normalised values,
- automation streams,
- gesture updates,
- parameter change notifications.

Signal MUST NOT perform:

- parameter name lookups,
- semantic reasoning,
- mapping or aliasing,
- non-RT metadata loading.

Signal treats parameters as numeric identifiers and values only.

---

## 10. Future Extensions

The parameter system is designed for extensibility. Future developments may include:

- dynamic parameter sets (modular/plugin-generated),
- parameter namespaces per lane or Clip,
- AI-assisted semantic grouping,
- cross-project parameter macros,
- modulation matrix as first-class architecture.
