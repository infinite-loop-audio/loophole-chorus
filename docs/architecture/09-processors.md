# Processors Architecture

This document defines the conceptual model for **processors** in Loophole. It
covers what a processor is, how processors are identified and owned, how they
form Channel chains, how they relate to Tracks, Clips and Lanes, and how they
interact with parameters, automation, Pulse, Signal and Composer.

This is an architectural document. It describes shapes, responsibilities and
relationships, not JUCE implementation details or IPC formats.

---

## Contents

- [1. Overview](#1-overview)
- [2. Processor Responsibilities](#2-processor-responsibilities)
- [3. Processor Types](#3-processor-types)
  - [3.1 Instruments](#31-instruments)
  - [3.2 Audio Effects](#32-audio-effects)
  - [3.3 LaneStream Processors](#33-lanestream-processors)
  - [3.4 Utility and Analysis Processors](#34-utility-and-analysis-processors)
  - [3.5 Device/IO Processors](#35-deviceio-processors)
- [4. Processor Identity](#4-processor-identity)
  - [4.1 Processor IDs](#41-processor-ids)
  - [4.2 Plugin Identity vs Processor Identity](#42-plugin-identity-vs-processor-identity)
  - [4.3 Stability Across Edits](#43-stability-across-edits)
- [5. Processor Chains and Channels](#5-processor-chains-and-channels)
  - [5.1 Serial Chains](#51-serial-chains)
  - [5.2 Relationship to Tracks and Clips](#52-relationship-to-tracks-and-clips)
  - [5.3 LaneStreams as First-Class Processors](#53-lanestreams-as-first-class-processors)
- [6. Parameters and Processors](#6-parameters-and-processors)
  - [6.1 Parameter Ownership](#61-parameter-ownership)
  - [6.2 Parameter Discovery](#62-parameter-discovery)
  - [6.3 Automation Targets](#63-automation-targets)
- [7. Processor Lifecycle](#7-processor-lifecycle)
  - [7.1 Creation](#71-creation)
  - [7.2 Replacement](#72-replacement)
  - [7.3 Removal](#73-removal)
  - [7.4 Bypassing](#74-bypassing)
- [8. Interaction with Pulse](#8-interaction-with-pulse)
- [9. Interaction with Signal](#9-interaction-with-signal)
- [10. Interaction with Composer](#10-interaction-with-composer)
- [11. Interaction with Aura](#11-interaction-with-aura)
- [12. Future Extensions](#12-future-extensions)

---

## 1. Overview

Processors are the core building blocks of audio and control processing in
Loophole. They live inside **Channels** and are executed by **Signal** in a
strictly serial chain. Processors include:

- instruments,
- audio effects,
- LaneStream nodes,
- utility and analysis nodes,
- device/IO abstractions.

Processors:

- own parameters,
- receive audio, MIDI or other control streams,
- produce audio and/or control streams,
- participate in automation and parameter mapping.

Pulse manages processor chains as part of the project model. Signal instantiates
and runs processors. Aura visualises and edits them. Composer enriches
processor-related metadata but never controls processor behaviour directly.

---

## 2. Processor Responsibilities

A processor:

- consumes zero or more input streams (audio, MIDI, control),
- produces zero or more output streams (audio, MIDI, control),
- exposes a set of parameters,
- responds to parameter changes and automation,
- performs a deterministic transformation on its inputs.

Processors must not:

- own project-level state,
- store Clip or Track structures,
- perform non-deterministic transformations dependent on UI state,
- manage their own routing beyond their defined inputs/outputs.

---

## 3. Processor Types

### 3.1 Instruments

Instrument processors:

- receive MIDI (and optionally other control streams),
- produce audio,
- may expose large parameter sets,
- may support MPE, per-note expressions and other advanced control schemes.

An instrument is just a processor from the engine’s point of view. The key
distinction is the presence of MIDI input and audio output.

### 3.2 Audio Effects

Audio effect processors:

- receive audio (and optionally MIDI/control),
- produce audio,
- may be pre- or post-fader (depending on chain position),
- include EQs, dynamics, saturators, reverbs, delays and so on.

Effects may also expose sidechain inputs; those are represented as additional
inputs in the processor model.

### 3.3 LaneStream Processors

LaneStream processors are special-purpose processors that:

- receive audio from Clip audio Lanes on a Track,
- appear at (or near) the top of the Channel processor chain by default,
- provide per-Lane mixing controls (gain, pan, phase, mute, etc.),
- may be split so different sets of Lanes feed different LaneStreams.

LaneStream processors are first-class processors:

- they have parameters,
- they can be automated,
- they can be reordered (subject to constraints defined by Pulse),
- they can be inspected in Aura like any other processor.

### 3.4 Utility and Analysis Processors

Utility processors include:

- gain and trim nodes,
- pan and balance nodes,
- meter nodes,
- channel format converters (mono/stereo/etc.).

Analysis processors include:

- spectrum analysers,
- loudness meters,
- correlation meters.

These processors may be native to Signal or plugin-based. In both cases they
obey the same processor contract.

### 3.5 Device/IO Processors

Device/IO processors represent:

- audio interface inputs and outputs,
- hardware insert send/return points,
- external device control endpoints.

They are the bridge between the abstract engine graph and the actual system
hardware and device layer.

---

## 4. Processor Identity

### 4.1 Processor IDs

Each processor has a stable ID within its Channel:

- assigned by Pulse on creation,
- unique per Channel,
- stable across project saves, undo/redo, and minor edits.

Processor IDs are used in:

- parameter IDs (see [@chorus:/docs/architecture/08-parameters.md](08-parameters.md)),
- automation targets,
- routing specifications,
- graph update messages to Signal.

### 4.2 Plugin Identity vs Processor Identity

A processor may be:

- a plugin instance (VST3, CLAP, AU), or
- a built-in DSP unit (utility, LaneStream, device IO).

**Plugin identity** refers to:

- vendor/product,
- format,
- version.

**Processor identity** refers to:

- this specific instance in this Channel in this project.

Multiple processors can share the same plugin identity (several instances of the
same plugin in different places), but they always have distinct processor IDs.

### 4.3 Stability Across Edits

Processor IDs MUST remain stable under:

- Track duplication,
- Clip editing,
- simple reordering of processors,
- parameter edits.

Processor IDs MAY change under:

- Channel reallocation across engines (future),
- certain destructive edits that fundamentally replace processors.

Pulse is responsible for ensuring that automation and parameter bindings are
updated if processor IDs are changed.

---

## 5. Processor Chains and Channels

### 5.1 Serial Chains

Each Channel has a serial processor chain:

```
[ LaneStream(s) ] → [ Instruments ] → [ Effects ] → [ Utility/Metering ] → [ Output ]
```

There is no concurrent parallel graph at this level; parallelism is achieved via:

- LaneStreams feeding different processors,
- multiple Channels,
- sends/returns (future),
- routing and summing between Channels.

Pulse defines the chain; Signal executes it.

### 5.2 Relationship to Tracks and Clips

Processors:

- live in Channels,
- do not live in Tracks or Clips.

Tracks:

- reference Channels,
- cause Channels to exist (by requiring audio/instruments),
- never embed processors directly.

Clips:

- define Lanes and content,
- feed LaneStreams and processors via Channel routing,
- never own processors.

### 5.3 LaneStreams as First-Class Processors

LaneStreams are treated as processors in the chain:

- they may appear at different positions in the chain,
- they can be moved before or after specific effect nodes (subject to rules),
- they expose their own parameters for:

  - per-Lane gain/pan/phase,
  - summing modes,
  - possibly per-Lane timing/phase alignment (future).

This allows complex workflows such as:

- processing only certain Lane outputs before they are mixed with others,
- inserting effects that see only a subset of Lanes,
- creating internal “submixes” within a Track.

---

## 6. Parameters and Processors

### 6.1 Parameter Ownership

Processors own parameters. For each processor:

- all of its parameters are attached to that processor ID,
- parameter IDs are processor-local and combined with the processor ID to form
  fully qualified parameter IDs,
- Pulse is responsible for reflecting these parameters in the global parameter
  model.

LaneStream parameters are also processor-owned:

- per-Lane level, pan, mute, phase, etc.,
- aggregated or per-Lane views for automation.

### 6.2 Parameter Discovery

When a processor is created or replaced:

- Signal queries the plugin/DSP node for its parameters,
- Pulse receives the parameter list,
- Pulse merges this with existing alias tables and Composer metadata,
- Pulse exposes the resolved parameter set to Aura.

Parameter discovery must be deterministic for a given plugin/DSP implementation.

### 6.3 Automation Targets

Automation Lanes target processor parameters through:

- Channel ID,
- processor ID,
- parameter ID.

Automation is scoped at the Clip level but resolves to processor parameters at
playback. Processor replacement and plugin updates may change the available
parameters; Pulse must remap automation where possible.

---

## 7. Processor Lifecycle

### 7.1 Creation

Processors can be created by:

- user inserting a plugin or built-in processor in Aura,
- implicit creation (LaneStream when first audio lane appears),
- Track templates and presets,
- project loading.

Pulse adds a processor to the Channel chain, assigns an ID, and instructs Signal
to instantiate it.

### 7.2 Replacement

Processor replacement occurs when:

- a user swaps one plugin for another,
- a plugin format is changed (VST3 → CLAP),
- a plugin is updated and old binary is no longer available,
- a processor is upgraded to a different implementation.

Pulse:

- creates a new processor instance,
- reassigns automation via alias and Composer metadata where possible,
- updates the Channel chain accordingly.

Signal is instructed to tear down the old processor and load the new one.

### 7.3 Removal

Removing a processor:

- removes it from the Channel chain,
- invalidates its parameters,
- may orphan automation lanes (Pulse should warn and offer reassignment),
- may necessitate rebalancing levels or adjusting Lane routing.

Removed processors are not kept in memory; state persists only via project
serialisation or explicit user actions (e.g. presets).

### 7.4 Bypassing

Processors may be bypassed:

- bypass is a parameter state, not structural removal,
- bypass should be automatable (subject to performance constraints),
- bypass semantics should be consistent across plugin formats where possible.

Bypass is applied in Signal but controlled and persisted by Pulse.

---

## 8. Interaction with Pulse

Pulse:

- owns the processor chain model for each Channel,
- creates, reorders, replaces and removes processors,
- manages processor IDs and mapping,
- owns parameter state and automation,
- determines how LaneStreams are created and routed,
- uses Composer to interpret plugin and parameter semantics.

Processors never directly “talk” to Pulse; all communication is mediated through
Signal and the processor/parameter model.

---

## 9. Interaction with Signal

Signal:

- instantiates processor implementations (plugins/DSP),
- manages plugin lifecycles (initialise, prepare, process, release),
- applies parameter updates at audio rate,
- runs processor chains in real time,
- reports failures or status changes back to Pulse.

Signal does not interpret:

- semantic roles,
- aliasing,
- routing rules beyond execution,
- plugin categorisation.

All those concerns are handled by Pulse (and Composer).

---

## 10. Interaction with Composer

Processors themselves are not aware of Composer. Composer metadata affects how
Pulse reasons about processors:

- plugin identity normalisation (family, variants, formats),
- parameter metadata (names, ranges, units, scaling),
- semantic roles (e.g. `filter.cutoff`, `fx.mix`),
- mapping suggestions when replacing or updating processors.

Composer helps Pulse preserve processor behaviour at the project level when the
underlying plugin or DSP implementation changes.

---

## 11. Interaction with Aura

Aura:

- displays processor chains in the mixer and Track inspector,
- allows inserting, reordering, bypassing and removing processors,
- opens plugin UIs (via Signal window handles),
- exposes LaneStream controls,
- surfaces Composer-enriched metadata (parameter roles, groups, categories).

Aura never directly controls processor execution or routing. It sends editing
intents to Pulse and renders the resulting model.

---

## 12. Future Extensions

The processor model anticipates future capabilities such as:

- multi-channel and spatial processors,
- graph-based internal routing within a processor “slot” (e.g. container
  processors),
- macro processors that wrap multiple internal processors,
- dynamic processor graphs per Clip or Section (with strict RT constraints),
- per-Lane insert chains (inside LaneStreams),
- distributed or remote processors.

All such features must preserve the core invariants:

- Pulse owns processor structure and identity,
- Signal executes a resolved serial graph,
- Aura reflects, not invents, processor configuration,
- Composer remains a metadata-only service.
