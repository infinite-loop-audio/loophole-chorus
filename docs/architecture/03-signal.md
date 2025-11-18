# Signal Architecture

Signal is the real-time audio engine of Loophole. It runs as an independent,
high-performance process responsible for deterministic audio execution, plugin
hosting, real-time parameter application and delivering sample-accurate output.

Signal performs *no* project management, editing logic, metadata interpretation,
or structural inference. Pulse constructs the engine graph; Signal executes it.

---

## Contents

- [1. Overview](#1-overview)
- [2. Responsibilities](#2-responsibilities)
- [3. Process Model](#3-process-model)
  - [3.1 Real-Time Constraints](#31-real-time-constraints)
  - [3.2 Multi-Threading Model](#32-multi-threading-model)
  - [3.3 Memory Management Rules](#33-memory-management-rules)
- [4. Engine Graph](#4-engine-graph)
  - [4.1 Graph Structure](#41-graph-structure)
  - [4.2 Nodes](#42-nodes)
  - [4.3 Node Order](#43-node-order)
  - [4.4 LaneStreamNode](#44-lanestreamnode)
  - [4.5 Graph Updates](#45-graph-updates)
- [5. Plugin Hosting](#5-plugin-hosting)
  - [5.1 Formats](#51-formats)
  - [5.2 Plugin Isolation](#52-plugin-isolation)
  - [5.3 UI Windows](#53-ui-windows)
- [6. Parameter Application](#6-parameter-application)
  - [6.1 Registered Parameters](#61-registered-parameters)
  - [6.2 Automation Streams](#62-automation-streams)
  - [6.3 Gesture Streams](#63-gesture-streams)
  - [6.4 Value Smoothing](#64-value-smoothing)
- [7. Transport](#7-transport)
  - [7.1 Clock and Timeline](#71-clock-and-timeline)
  - [7.2 Playback State](#72-playback-state)
  - [7.3 Synchronisation](#73-synchronisation)
- [8. Interaction with Pulse](#8-interaction-with-pulse)
  - [8.1 Structural Commands](#81-structural-commands)
  - [8.2 Parameter and Automation Commands](#82-parameter-and-automation-commands)
  - [8.3 High-Rate Control Path](#83-high-rate-control-path)
  - [8.4 Engine Status Events](#84-engine-status-events)
- [9. Interaction with Composer](#9-interaction-with-composer)
- [10. Persistence](#10-persistence)
- [11. Processing Cohorts and Anticipative Rendering](#11-processing-cohorts-and-anticipative-rendering)
- [12. Hardware I/O and Control Surfaces](#12-hardware-io-and-control-surfaces)
  - [12.1 Device Enumeration and Discovery](#121-device-enumeration-and-discovery)
  - [12.2 Input Parsing and Forwarding](#122-input-parsing-and-forwarding)
  - [12.3 Output Dispatch](#123-output-dispatch)
  - [12.4 Transport Control Path](#124-transport-control-path)
  - [12.5 Gesture and Value-Stream Participation](#125-gesture-and-value-stream-participation)
- [13. Future Extensions](#13-future-extensions)

---

## 1. Overview

Signal is a dedicated audio engine responsible for:

- deterministic audio processing,
- hosting plugins and DSP units,
- processing automation and gestures,
- running transport,
- delivering final audio output.

Signal receives a fully resolved, validated engine graph from Pulse. It does not
interpret or modify project structure.

Signal is written in C++ (JUCE-based foundation). It must remain small,
focused and real-time safe.

---

## 2. Responsibilities

Signal handles:

1. **Plugin hosting**
   Loading, initialising and running instrument and effect plugins.

2. **Real-time DSP**
   Executing nodes sample-accurately.

3. **Parameter updates**
   Applying automation and gestures at audio-rate.

4. **Transport**
   Maintaining playback position, tempo, sync and sample clocks.

5. **Audio I/O**
   Capturing and rendering audio to hardware.

6. **Engine graph execution**
   Running the node graph constructed by Pulse.

7. **Hardware device enumeration and I/O**
   Discovering and managing MIDI, HID, OSC, and audio devices; parsing low-level inputs; sending hardware outputs.

Signal does *not* manage routing rules, editing operations, metadata resolution, or mapping decisions for control surfaces.

---

## 3. Process Model

### 3.1 Real-Time Constraints

Signal must:

- perform no dynamic allocation in the audio thread,
- avoid locks, system calls, or blocking IPC in the audio thread,
- keep all plugin calls within deterministic time limits,
- never parse JSON or perform text processing during audio callbacks.

### 3.2 Multi-Threading Model

Signal uses:

- a high-priority audio thread (non-negotiable real-time constraints),
- a message thread for IPC, plugin scanning, parameter staging,
- optional worker threads for background tasks.

All IPC messages from Pulse are processed outside the audio thread.

### 3.3 Memory Management Rules

- All buffers pre-allocated.
- No heap usage during processing.
- All plugin allocations occur at load/prepare time.

---

## 4. Engine Graph

### 4.1 Graph Structure

Pulse sends Signal a linear node graph. Signal does not build or modify the
graph; it only prepares and executes it.

### 4.2 Nodes

Node types include:

- **LaneStreamNode(s)** (often shortened to 'LaneStream') (audio input from Tracks),
- **Instruments**,
- **Audio effects**,
- **Utility DSP** (gain, pan, meters),
- **Output nodes**.

### 4.3 Node Order

Order is strictly serial. Signal executes nodes in the order defined by Pulse.

### 4.4 LaneStreamNode

LaneStreamNode(s) (often shortened to 'LaneStream') provide a buffer for audio from Clip Lanes routed via Pulse. Signal treats them as input sources. They are:

- created/destroyed by graph updates,
- filled with audio data from Pulse-provided buffers or internal feeders,
- processed as standard audio nodes.

### 4.5 Graph Updates

Pulse may send:

- node addition/removal,
- reordering instructions,
- instrument replacement,
- routing updates.

Signal must:

- commit these changes outside the audio thread,
- prepare any new nodes before activation,
- swap to the new graph atomically and safely.

---

## 5. Plugin Hosting

### 5.1 Formats

Signal supports:

- VST3,
- CLAP,
- AU (platform-dependent).

Other formats can be added later.

### 5.2 Plugin Isolation

Signal may optionally support:

- per-plugin sandboxing,
- crash recovery,
- watchdog monitoring.

These features must never degrade real-time performance.

### 5.3 UI Windows

Signal hosts plugin UIs as native windows and hands their handles to Aura.

Signal takes no part in UI layout, styling or event management beyond OS-level window embedding.

---

## 6. Parameter Application

### 6.1 Registered Parameters

Pulse registers each parameter with Signal:

- parameter ID,
- target node,
- scaling/normalisation,
- smoothing behaviour.

Signal stores this in a fast lookup table.

### 6.2 Automation Streams

Automation from Pulse arrives as timestamped value updates. Signal:

- applies them at audio-rate,
- interpolates if required,
- guarantees ordering and determinism.

### 6.3 Gesture Streams

High-rate gestures bypass automation evaluation.
Signal receives gesture packets on a separate fast channel and applies them directly.

### 6.4 Value Smoothing

Signal provides built-in smoothing for:

- gain,
- filter parameters,
- pitch parameters,
- any parameter with defined smoothing rules.

Smoothing specifications come from Pulse metadata.

---

## 7. Transport

### 7.1 Clock and Timeline

Signal maintains:

- sample counter,
- beat/bar position (derived),
- tempo and time signature (from Pulse),
- loop boundaries.

### 7.2 Playback State

Playback states include:

- stopped,
- playing,
- paused,
- seeking.

Transport changes must not interrupt or reallocate DSP nodes except as instructed.

### 7.3 Synchronisation

Signal ensures:

- sample-accurate timing of events,
- stable transport at all buffer sizes,
- tight sync with external devices (future).

---

## 8. Interaction with Pulse

### 8.1 Structural Commands

Signal receives commands for:

- loading/unloading nodes,
- reordering nodes,
- adding LaneStreamNodes,
- removing nodes,
- updating routing,
- resending full engine graph when required.

### 8.2 Parameter and Automation Commands

Pulse sends:

- parameter value updates,
- automation curves,
- modulation routing (future).

Signal applies them without reinterpreting or validating semantics.

### 8.3 High-Rate Control Path

Gesture updates are delivered via a dedicated low-latency channel. Signal applies
them immediately to plugin nodes.

### 8.4 Engine Status Events

Signal emits back to Pulse:

- plugin load failures,
- processing errors (with safe fallback),
- plugin crashes (if isolation is enabled),
- transport state confirmations.

---

## 9. Interaction with Composer

Signal **never** interacts with Composer.

Signal receives already-resolved node identities, parameter targets and roles
from Pulse. All metadata interpretation is complete before Signal receives the
instruction.

---

## 10. Persistence

Signal holds no persistent state. All persistent representation lives in Pulse.

Signal may cache:

- plugin binaries,
- plugin scanning results,
- DSP configurations,

but none of these form part of the project state.

---

## 11. Processing Cohorts and Anticipative Rendering

Loophole's engine supports two processing domains: **live** and **anticipative** processing cohorts (PC).
Signal runs two complementary processing engines to handle these cohorts efficiently
and maintain both low-latency real-time responsiveness and maximum CPU efficiency.

### 11.1 Live Engine

The **Real-Time Engine** runs in the audio callback and executes all live cohort
nodes:

- record-armed tracks,
- instrument tracks receiving live MIDI,
- plugins with GUIs open,
- nodes marked non-deterministic,
- send/return busses feeding live nodes,
- any downstream nodes reachable from the above.

The live engine operates at short buffer sizes (64–128 samples) to maintain
low-latency, sample-accurate processing. It pulls anticipative outputs from timeline
buffers where required, applies gesture updates and real-time automation, and
prioritises stability and responsiveness above all else.

### 11.2 Anticipative Engine

The **Anticipative Engine** runs on worker threads and processes ANTICIPATIVE cohort
nodes far ahead of the playhead. These nodes are deterministic and have no
real-time dependency:

- nodes that are deterministic,
- nodes not downstream of any live nodes,
- nodes that have no GUI open and are not receiving live input.

The anticipative engine uses very large buffers (hundreds of ms to several seconds)
for maximum CPU efficiency. It maintains a **render horizon buffer** representing
multiple seconds ahead of the current playhead position, effectively functioning like
continuous, invisible “background freeze”.

### 11.3 Render Horizon

The render horizon buffer maintains pre-rendered audio for anticipative nodes
ahead of the playhead. Signal:

- maintains a buffer representing playhead position + multiple seconds,
- continuously renders anticipative nodes into this buffer,
- invalidates portions when live gestures or parameter changes override them,
- ensures the horizon remains far enough ahead to avoid underruns,
- synchronises automation, modulation and tempo with Pulse.

The render horizon must be managed carefully to balance memory usage with the need
for sufficient lookahead.

### 11.4 Buffer Model

Signal maintains separate buffer pools:

- **Real-time buffers** (small, frequently recycled for live processing),
- **Anticipative buffers** (large blocks optimised for batch processing),
- **Timeline buffers** (read-only, pre-rendered anticipative audio),
- **Crossfade buffers** (temporary buffers used during cohort transitions).

All buffers are pre-allocated; no dynamic allocation occurs during processing.

### 11.5 Cohort Transitions

When a node changes cohort assignment (due to routing changes, GUI open/close,
arming state, or user override), Signal must handle the transition smoothly:

- **Preparing target domain**: Signal prepares the node state in the target
  domain (live or anticipative) before activation.

- **Crossfading**: Signal crossfades between anticipative and live audio where
  required to avoid clicks or discontinuities.

- **Buffer invalidation**: Signal invalidates anticipative buffers when live gestures
  override them, forcing re-render of affected regions.

- **Domain synchronisation**: Signal ensures no gap or glitch occurs when switching
  domains, maintaining sample-accurate timing throughout.

- **Graph updates**: Signal receives updated graph instructions from Pulse reflecting
  new cohort assignments and commits them atomically.

### 11.6 Thread Responsibilities

Signal’s threading model supports both domains:

- **Audio thread**: Executes live engine processing, pulls from timeline buffers,
  applies gestures and real-time automation. Must never block, allocate, or perform
  non-deterministic operations.

- **Worker threads**: Execute anticipative engine processing, render into horizon
  buffers, process large blocks efficiently. May perform more CPU-intensive
  operations but must synchronise state with audio thread.

- **Message thread**: Handles IPC from Pulse, processes graph updates, coordinates
  cohort transitions, manages plugin UI lifecycle.

Signal ensures both domains remain sample-accurate and aligned, with proper
synchronisation between threads to avoid race conditions or timing drift.

---

## 12. Hardware I/O and Control Surfaces

Signal is responsible for low-level hardware I/O operations and acts as a transport layer for control surface communication.

### 12.1 Device Enumeration and Discovery

Signal handles:

- enumerating hardware devices (MIDI, HID, OSC, HUI/MCU),
- detecting plug/unplug events,
- classifying endpoints and protocols,
- providing device descriptors to Pulse with timestamps.

Signal never makes mapping decisions; it only reports what hardware is available and what events occur.

### 12.2 Input Parsing and Forwarding

Signal:

- parses low-level inputs (MIDI messages, HID reports, OSC packets),
- forwards structured **control messages** to Pulse with timestamps,
- handles protocol-specific parsing (sysex identity queries, HID descriptor decoding),
- ensures low-latency forwarding (within a few milliseconds).

Signal **does not** interpret what a control does; it only reports "fader 1 moved" or "pad 5 pressed", not "change track 5 volume" or "trigger clip".

### 12.3 Output Dispatch

Signal receives **Feedback Intents** from Pulse and translates them into hardware outputs:

- MIDI CC messages for LEDs,
- RGB pad commands,
- sysex display updates,
- MCU/HUI display messages,
- HID output reports.

Signal's job is purely delivery; Pulse decides all behaviour and visual feedback.

### 12.4 Transport Control Path

Transport controls (play/stop/record) may have a low-latency path via Signal, but Pulse still owns the authoritative transport state. Signal forwards transport button presses to Pulse and executes transport commands from Pulse.

### 12.5 Gesture and Value-Stream Participation

Signal participates in gesture/value-stream IPC for high-resolution controller movements. Control-surface inputs that are high-resolution or continuous can participate in gesture/value-stream sessions, allowing Pulse to distinguish between:

- high-res *gesture streams* (e.g. knob sweeps, fader movements),
- normal control events (e.g. simple button presses).

Signal forwards gesture stream data to Pulse and applies gesture-driven parameter updates in the audio engine.

---

## 13. Testing & Validation

Signal's architecture includes testing and validation as a core design concern:

- **Graph-level tests**:
  - Verifying node ordering and routing correctness,
  - Cohort transitions and dependency closure,
  - Graph stability under structural changes.

- **Render tests**:
  - Offline renders compared against golden reference outputs,
  - Ensuring deterministic behaviour where required,
  - Verifying sample-accurate timing and synchronisation.

- **Realtime safety harness**:
  - Tests that stress timing and buffer handling,
  - Verifying no allocations or blocking calls occur on the audio thread,
  - Validating that all real-time constraints are met under load.

- **Plugin hosting tests**:
  - Plugin scanning and discovery,
  - Sandboxing and crash recovery scenarios,
  - Plugin UI lifecycle and window management,
  - Parameter application and automation correctness.

- **Hardware I/O tests**:
  - Device enumeration and discovery,
  - Input parsing and forwarding correctness,
  - Output dispatch and feedback intent handling.

Testing ensures that Signal maintains real-time safety, that plugin hosting is robust, and that the engine graph executes correctly under all conditions.

---

## 14. Future Extensions

Signal may evolve to support:

- multi-engine distributed processing,
- GPU-accelerated DSP nodes,
- per-plugin sandboxes,
- hybrid offline/real-time processing,
- real-time stem bounces,
- remote/networked DSP.

These must not affect the core contract: Pulse owns structure; Signal executes.
