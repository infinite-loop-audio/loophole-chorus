# Parameters Architecture

This document defines the conceptual model for parameters in Loophole. It covers
parameter identity, ownership, metadata, automation targeting, propagation
between Aura, Pulse and Signal, and how parameter remapping and aliasing work
when plugins or processors change.

This is an architectural document. It describes shapes and responsibilities, not
wire formats or IPC details.

---

## Contents

- [1. Overview](#1-overview)
- [2. Parameter Identity](#2-parameter-identity)
  - [2.1 Fully Qualified Parameter IDs](#21-fully-qualified-parameter-ids)
  - [2.2 Aliasing and Remapping](#22-aliasing-and-remapping)
  - [2.3 Format Switching](#23-format-switching)
- [3. Parameter Ownership and Hosting](#3-parameter-ownership-and-hosting)
  - [3.1 Processor Parameters](#31-processor-parameters)
  - [3.2 Channel Parameters](#32-channel-parameters)
  - [3.3 Track Parameters](#33-track-parameters)
  - [3.4 Device Parameters](#34-device-parameters)
- [4. Parameter Types and Metadata](#4-parameter-types-and-metadata)
  - [4.1 Core Types](#41-core-types)
  - [4.2 Metadata Fields](#42-metadata-fields)
  - [4.3 Scaling and Presentation](#43-scaling-and-presentation)
  - [4.4 Parameter Grouping](#44-parameter-grouping)
- [5. Automation Targeting](#5-automation-targeting)
  - [5.1 One Lane, One Primary Parameter](#51-one-lane-one-primary-parameter)
  - [5.2 Linked Parameters (Future)](#52-linked-parameters-future)
- [6. Parameter Stability and Lifecycle](#6-parameter-stability-and-lifecycle)
  - [6.1 Stable IDs per Processor Instance](#61-stable-ids-per-processor-instance)
  - [6.2 Plugin Updates and Versioning](#62-plugin-updates-and-versioning)
- [7. Parameter Propagation](#7-parameter-propagation)
  - [7.1 Aura ↔ Pulse](#71-aura--pulse)
  - [7.2 Pulse ↔ Signal](#72-pulse--signal)
  - [7.3 High-Rate vs Low-Rate Updates](#73-high-rate-vs-low-rate-updates)
- [8. Interaction with Lanes and Automation](#8-interaction-with-lanes-and-automation)
- [9. Future Extensions](#9-future-extensions)

---

## 1. Overview

Parameters represent all continuous and discrete controls in Loophole:

- plugin controls (cutoff, drive, feedback),
- LaneStream controls (per-lane gain/pan/phase),
- channel controls (fader, pan, mute, solo),
- track-level control flags (arm, monitor modes),
- device parameters (monitor controller trims, hardware insert gains),
- internal engine controls.

Parameters exist conceptually in Pulse, are executed in Signal, and are
presented and edited in Aura. The design aims to:

- keep parameter identity stable across sessions,
- allow plugins and processors to evolve without breaking projects,
- support automation, modulation and device control,
- support future switching between plugin formats where possible.

---

## 2. Parameter Identity

### 2.1 Fully Qualified Parameter IDs

Loophole uses fully qualified parameter IDs. Conceptually, a parameter is
addressed by a structured path:

- `channelId` — which Channel,
- `processorId` — which processor within that Channel (instrument, FX, LaneStream, etc.),
- `parameterId` — which parameter within the processor.

Additional contexts (such as device or global parameters) extend this scheme by
introducing:

- a well-known `channelId` for global or device contexts, or
- a different host type identifier mapped to the same conceptual model in Pulse.

Pulse treats fully qualified IDs as the canonical reference for:

- automation lanes,
- UI bindings,
- device mappings,
- project persistence.

Aura does not need to expose full IDs to users; it uses them internally for
routing and correlation.

### 2.2 Aliasing and Remapping

Because processors (especially third-party plugins) can change over time,
Loophole includes an explicit **parameter aliasing and remapping layer** in
Pulse.

Pulse maintains:

- a primary mapping from persisted parameter IDs → current parameter
  definitions,
- an alias table for mapping older or alternate IDs to current ones.

On project load:

- Pulse attempts to resolve each stored parameter ID to a current parameter.
- If the exact ID exists, it is used directly.
- If it does not, Pulse can:
  - consult alias mappings (e.g. plugin-provided or previously learned),
  - attempt heuristic matching (by name, type, range, etc.),
  - mark unresolved parameters for user attention.

Aura may present a **remapping UI** when mappings are ambiguous, allowing users
to choose how old parameters should map to new ones. Once confirmed, Pulse can
persist these remapping rules as aliases.

This model aims to prevent silent loss of automation and control when plugins
are updated or reinstalled.

### 2.3 Format Switching

Loophole’s parameter model is designed to make it possible to switch between
different plugin formats (for example, VST and CLAP instances of the same
plugin) without discarding parameter and automation data.

In this scenario:

- Pulse treats different plugin formats as different processor implementations
  that may expose equivalent parameter sets.
- Plugins can provide metadata indicating that two processor instances (e.g.
  `MySynth VST` and `MySynth CLAP`) share a common logical parameter namespace.
- Pulse’s aliasing and remapping layer can use this information to remap
  parameters when switching formats.

The detailed format switching workflow is defined in later documents. This
document establishes that parameter identity, aliasing and remapping are
architected to support this capability.

---

## 3. Parameter Ownership and Hosting

### 3.1 Processor Parameters

Processors are the primary owners of parameters. This includes:

- instruments and audio effects,
- MIDI processors,
- LaneStream nodes,
- internal DSP units in Signal.

Each processor:

- exposes a set of parameters with stable IDs (for the lifetime of that
  instance),
- describes these parameters via the metadata model in Pulse,
- receives value updates via Pulse → Signal communication.

### 3.2 Channel Parameters

Channels own parameters that relate to channel-level behaviour, such as:

- fader gain,
- pan,
- mute/solo (engine-side state),
- input/output configuration and trims.

These parameters are represented in Pulse with fully qualified IDs that reflect
their presence on the Channel rather than on a specific child processor.

### 3.3 Track Parameters

Tracks have parameters that are purely model/UI concepts, such as:

- track mute/solo/arm/monitor modes (the model flags),
- track height, visibility and other layout considerations.

Some track parameters may be mirrored into Channel parameters (for example,
track mute might be reflected as a channel mute). Others exist purely in Pulse
and Aura and never reach Signal.

Track parameters are owned and resolved by Pulse and are not visible in Signal.

### 3.4 Device Parameters

Device parameters describe:

- audio interface-level controls,
- external hardware device controls,
- monitor controller behaviour,
- other non-plugin, non-channel hardware state.

These parameters are owned by a device abstraction layer in Pulse. When they
have real-time consequences, they are reflected into Signal or into external
APIs as appropriate.

Device parameters are addressable through the same parameter identity and
metadata system, allowing them to be automated via Device Lanes and edited
within Aura.

---

## 4. Parameter Types and Metadata

### 4.1 Core Types

Parameters are strongly typed. Core types include:

- continuous scalar values (float),
- integers,
- booleans,
- enums (discrete symbolic states),
- time values (where a time-specific type is useful),
- level values (gain in dB or linear units).

Internal representations in Signal may be normalised or transformed; Pulse
maintains the canonical type and range.

### 4.2 Metadata Fields

Each parameter is described by a metadata block, including:

- `id` — processor-local parameter ID,
- `name` — human-readable name,
- `shortName` — a shorter label for compact UIs,
- `description` — optional longer explanation,
- `type` — core type (float, int, bool, enum, time, level),
- `min`, `max` — numeric range (where applicable),
- `default` — default value,
- `unit` — unit symbol or label (Hz, dB, %, ms, etc.),
- `step` — minimum useful resolution (for stepped controls),
- `flags`:
  - `automatable`,
  - `modulatable`,
  - `readOnly`,
  - `latent` (large smoothing or delay),
- `group` — grouping/category information (see below).

### 4.3 Scaling and Presentation

Parameters may have non-linear scaling for UI and automation:

- linear,
- logarithmic,
- exponential,
- custom curves,
- stepped/quantised behaviour.

Scaling is represented in metadata so that:

- Aura can choose appropriate widgets and display values meaningfully,
- automation curves can be generated and interpolated in a musically useful
  way,
- normalised internal values can be mapped to meaningful units.

The specific scaling representation (e.g. piecewise functions vs named curves)
is defined in later documents; this document requires that scaling be part of
the parameter model from the outset.

### 4.4 Parameter Grouping

Parameters can be grouped for presentation and navigation. Groups may represent:

- pages (e.g. “Oscillator”, “Filter”, “FX”),
- logical sections (e.g. “Input”, “Output”, “Mix”),
- device-specific categories.

Grouping is represented in metadata so Aura can:

- organise parameter browsers,
- build coherent plugin views,
- expose grouped automation assignment tools.

Grouping does not affect engine behaviour; it is a UI and workflow concern.

---

## 5. Automation Targeting

### 5.1 One Lane, One Primary Parameter

An automation Lane targets exactly one **primary** parameter:

- each automation Lane refers to a fully qualified parameter ID,
- all automation points in that Lane are interpreted as values for that
  parameter over time.

This model:

- keeps Pulse’s routing and evaluation simple and robust,
- makes automation mounting (attaching automation to different parameters)
  straightforward,
- simplifies editing tools in Aura.

### 5.2 Linked Parameters (Future)

It is useful in some workflows to control multiple parameters together (for
example, two filters moving in tandem). Rather than violating the “one lane,
one primary parameter” rule, Loophole may later introduce **linked
parameters**:

- a parameter may declare a linkage or follow relationship to another,
- or Pulse may allow users to define link groups mapping one automation source
  to multiple target parameters.

In this model, an automation Lane still targets a single primary parameter; it
is the parameter layer that fans out the effect to linked parameters. The
details of linked parameter groups are defined in a future document.

---

## 6. Parameter Stability and Lifecycle

### 6.1 Stable IDs per Processor Instance

Within a processor instance:

- parameter IDs remain stable for the lifetime of that instance,
- Pulse treats parameter IDs as opaque strings within the processor’s namespace
  but expects them not to change arbitrarily.

If a processor fundamentally changes its parameters (for example, loading a
completely different internal configuration), it should:

- either preserve existing parameter IDs where the semantics remain equivalent,
- or deliberately introduce new IDs that are distinct from old ones.

Pulse uses those IDs, together with aliasing, to maintain project integrity.

### 6.2 Plugin Updates and Versioning

When a plugin is updated or replaced:

- Signal may load a new processor implementation,
- Pulse re-queries the parameter set and compares it to stored definitions,
- Pulse uses:
  - exact ID matches,
  - alias mappings (where available),
  - heuristic matches (name, type, range, group),
to attempt to restore as many parameter mappings as possible.

Unresolved parameters are surfaced to Aura, which may provide:

- a mapping UI to allow manual reassignment,
- a view of which automation will not play back until resolved.

Once mappings are confirmed, Pulse can persist the new mappings as aliases for
future restores.

---

## 7. Parameter Propagation

### 7.1 Aura ↔ Pulse

In the Aura ↔ Pulse path:

- Aura sends **user intent**:
  - UI knob moves,
  - automation edits,
  - preset recalls,
  via identified parameters and values.
- Pulse:
  - validates parameter writes against metadata (range, type, flags),
  - updates its model state (where model-visible),
  - schedules updates to Signal (if the parameter is engine-visible),
  - emits model state changes back to Aura where relevant (e.g. when parameters
    are changed indirectly by other operations).

### 7.2 Pulse ↔ Signal

In the Pulse ↔ Signal path:

- Pulse sends parameter updates to Signal addressed by channel/processor and
  parameter IDs.
- Signal applies these updates to processors or internal controls.
- Signal may send parameter value reports back to Pulse when:
  - a processor changes its own state (e.g. randomise, preset load),
  - a device reports a new setting,
  - readback is necessary for UI coherence.

Pulse then reconciles these values with its model and updates Aura.

### 7.3 High-Rate vs Low-Rate Updates

Parameter updates are conceptually split into:

- **high-rate** streams:
  - automation playback,
  - continuous gestures,
  - modulation,
  where values may change at control rate or faster.

- **low-rate** events:
  - discrete UI operations (single knob change),
  - preset changes,
  - state loads,
  - device reconfiguration.

Loophole uses a hybrid strategy:

- High-rate updates are batched or streamed in a compact form suitable for the
  engine’s control rate and IPC bandwidth. Their exact streaming format is
  defined in transport and engine integration specifications.
- Low-rate updates are sent as discrete parameter change events over the
  control/model plane.

Pulse is responsible for choosing appropriate mechanisms based on context.

---

## 8. Interaction with Lanes and Automation

Automation Lanes inside Clips target parameters via their fully qualified IDs.
For any given automation Lane:

- Pulse uses the Lane’s target parameter to:
  - look up metadata and scaling,
  - evaluate the automation curve at playback time,
  - compute meaningful values and units.

Automation Lanes may exist:

- on Clips hosted by Tracks, targeting:
  - the Track’s Channel parameters,
  - processors in that Channel,
  - other Channels (automation mounting),
  - device parameters.

Global automation Lanes (e.g. part of global Device Lanes or other global
structures) similarly target parameters using the same identity and metadata
model.

---

## 9. Future Extensions

The parameters architecture is intended to support future additions such as:

- modulation sources and modulation matrixes,
- parameter morphing and snapshots,
- per-Clip parameter offsets,
- project-wide macros,
- advanced device control schemes.

These features will use the same identity, metadata and propagation model
described here.
