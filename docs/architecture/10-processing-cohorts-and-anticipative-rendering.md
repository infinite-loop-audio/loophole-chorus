# Processing Cohorts and Anticipative Rendering

This document defines the architecture for Loophole’s dual-domain audio
processing system: **Processing Cohorts**. Processing Cohorts allow the engine to
simultaneously deliver extremely low-latency interaction for live performance
while performing heavy DSP on large buffers in the background. This enables
Loophole to scale to projects that would otherwise exceed real-time CPU limits.

The design integrates with Pulse (model), Signal (DSP engine), Aura (UI) and
Composer (metadata), and forms a core part of how Loophole achieves
next-generation performance without requiring the user to manually freeze or
rebalance tracks.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Goals](#2-goals)  
- [3. Cohort Definitions](#3-cohort-definitions)  
  - [3.1 Live Cohort](#31-live-cohort)  
  - [3.2 Anticipative Cohort](#32-anticipative-cohort)  
- [4. Cohort Assignment (Pulse Responsibility)](#4-cohort-assignment-pulse-responsibility)  
  - [4.1 Determining Liveness](#41-determining-liveness)  
  - [4.2 Deterministic vs Non-Deterministic Processors](#42-deterministic-vs-non-deterministic-processors)  
  - [4.3 Dependency Closure](#43-dependency-closure)  
  - [4.4 User Overrides](#44-user-overrides)  
- [5. Execution Domains (Signal Responsibility)](#5-execution-domains-signal-responsibility)  
  - [5.1 Real-Time Engine](#51-real-time-engine)  
  - [5.2 Anticipative Engine](#52-anticipative-engine)  
  - [5.3 Render Horizon](#53-render-horizon)  
  - [5.4 Buffer Integration](#54-buffer-integration)  
- [6. Cohort Transitions](#6-cohort-transitions)  
  - [6.1 Live → Anticipative](#61-live--anticipative)  
  - [6.2 Anticipative → Live](#62-anticipative--live)  
  - [6.3 Crossfades and Safety](#63-crossfades-and-safety)  
- [7. Automation, Modulation and Gestures](#7-automation-modulation-and-gestures)  
- [8. Interaction with Tracks, Channels and Lanes](#8-interaction-with-tracks-channels-and-lanes)  
- [9. Interaction with Pulse](#9-interaction-with-pulse)  
- [10. Interaction with Composer](#10-interaction-with-composer)  
- [11. Interaction with Aura](#11-interaction-with-aura)  
- [12. Failure Modes and Fallbacks](#12-failure-modes-and-fallbacks)  
- [13. Future Extensions](#13-future-extensions)

---

## 1. Overview

Loophole separates DSP work into two parallel processing domains:

1. **Live Cohort**  
   - extremely low latency  
   - sample-accurate response to user input  
   - required for recording, playing instruments, manipulating parameters live  

2. **Anticipative Cohort**  
   - processed ahead of the playhead  
   - uses very large buffer sizes  
   - can effectively pre-render entire sections of the project in background  

This design allows Loophole to:

- handle much larger mixes than conventional DAWs,  
- keep MIDI instruments responsive even under high CPU load,  
- provide seamless “freeze-like” performance without manual freezing,  
- dynamically reassign processors as the user interacts.

---

## 2. Goals

Processing Cohorts enable:

1. **Scalability**  
   Heavy projects can run smoothly by pushing safe portions into anticipative rendering.

2. **Responsiveness**  
   Real-time performance is preserved regardless of project size.

3. **Transparency**  
   Users should not need to think about freezing, latency compensation or engine load.

4. **Predictability**  
   Automation, modulation and tempo changes remain sample-accurate in both domains.

5. **Dynamic Adaptation**  
   Opening a plugin UI or arming a track instantly moves that part of the graph into the live cohort.

---

## 3. Cohort Definitions

### 3.1 Live Cohort

Processors must be in the Live Cohort if they:

- depend on live audio input,  
- depend on live MIDI input,  
- feed a live processor via sends or group tracks,  
- have their plugin UI open,  
- contain non-deterministic DSP (randomised, chaotic, external input dependent),  
- have user-marked “Force Live”.

Live processors:
- run at the audio callback rate,  
- use short buffers (64–128 samples),  
- always respond instantly to gestures from Aura.

### 3.2 Anticipative Cohort

Processors may be in the Anticipative Cohort if they:

- are deterministic,  
- have no live inputs,  
- do not feed any live processor,  
- are explicitly marked “Prefer Pre-Render” by the user.

Anticipative processors:
- run on worker threads,  
- use large buffers (hundreds of ms to several seconds),  
- render into future timeline buffers (render horizon).

---

## 4. Cohort Assignment (Pulse Responsibility)

Pulse owns every decision regarding cohort membership. Signal must never guess.

### 4.1 Determining Liveness

Pulse inspects:

- track state (record-arm, monitoring),  
- processor metadata,  
- routing structure,  
- open UI windows (reported by Aura),  
- user preferences.

### 4.2 Deterministic vs Non-Deterministic Processors

Non-deterministic processors (e.g., analogue-modelled randomness, certain saturators):  
→ automatically live.

Deterministic processors:  
→ safe for anticipative rendering.

Composer may enrich this metadata, but Pulse is the authority.

### 4.3 Dependency Closure

If a processor is live, then:

- all **downstream** processors must also be live.  
- any **upstream node that provides live data** must be live.

This is a directed graph reachability rule.

### 4.4 User Overrides

Users may force a processor or track to:

- **Live only**,  
- **Prefer anticipative**,  
- **Automatic** (default).

Aura exposes these options; Pulse applies them when partitioning the graph.

---

## 5. Execution Domains (Signal Responsibility)

Signal runs both cohorts simultaneously.

### 5.1 Real-Time Engine

Handles:

- all Live Cohort processors,  
- gesture updates,  
- incoming MIDI and audio input,  
- sample-accurate parameter application.

Runs in the audio callback thread.

### 5.2 Anticipative Engine

Handles:

- all Anticipative Cohort processors,  
- automation and tempo-aware pre-render,  
- multi-second blocks.

Runs on worker threads.  
Must remain ahead of the playhead.

### 5.3 Render Horizon

Signal maintains a *render horizon time*:  

```
renderHorizon = playhead + anticipativeLeadTime
```

Where `anticipativeLeadTime` may range from ~0.5s to ~5s depending on CPU load.

Anticipative engine ensures all its outputs exist for every timestamp ≤ render horizon.

### 5.4 Buffer Integration

Live processors read anticipative outputs via:

- timeline-aligned circular buffers,  
- guaranteed sample alignment,  
- cross-domain parameter consistency.

---

## 6. Cohort Transitions

### 6.1 Live → Anticipative

Occurs when:

- track is unarmed,  
- UI for a plugin is closed,  
- deterministic state is confirmed.

Signal may continue using live output until anticipative renders overtake it.

### 6.2 Anticipative → Live

Occurs when:

- plugin UI opens,  
- track is armed,  
- live gesture occurs,  
- routing changes introduce live dependency.

Pulse sends a cohort update; Signal prepares the processor in the live chain.

### 6.3 Crossfades and Safety

When switching from anticipative to live:

- anticipative audio is used until live catches up,  
- a short crossfade prevents clicks,  
- anticipative buffers are invalidated and scheduled for re-render.

---

## 7. Automation, Modulation and Gestures

- Automation is applied identically in both domains.  
- Live gestures override anticipative rendering and may trigger cohort switches.  
- Modulation (LFOs, envelopes, MPE, etc.) uses the same temporal model in both domains.

---

## 8. Interaction with Tracks, Channels and Lanes

- Track flags influence cohort selection.  
- LaneStream processors inherit cohort membership from the Channel they belong to.  
- Audio Lanes pre-rendered in the anticipative domain behave similarly to track freezing.  
- Processor order changes may require temporary rebalancing of cohort membership.

---

## 9. Interaction with Pulse

Pulse:

- chooses cohorts,  
- issues graph updates,  
- recalculates dependency closure,  
- triggers transitions.

Signal executes those decisions without reinterpretation.

---

## 10. Interaction with Composer

Composer is metadata-only.  
It contributes deterministic/safety metadata, but:

- Signal never communicates with Composer,  
- Pulse is solely responsible for interpreting Composer’s metadata.

---

## 11. Interaction with Aura

Aura displays:

- cohort state per track/channel,  
- render progress (optional),  
- warnings when render horizon is close to playhead,  
- user controls for forcing live or anticipative mode.

User does not need to understand the engine internals for normal workflows.

---

## 12. Failure Modes and Fallbacks

If anticipative engine falls behind:

- Signal automatically promotes affected processors to the live cohort,  
- Pulse is notified,  
- CPU load indication is shown in Aura,  
- anticipative lead time is temporarily reduced.

If live engine overloads:

- Aura warns the user,  
- Pulse may demote some processors to anticipative mode if safe,  
- background rendering may be throttled.

---

## 13. Future Extensions

Potential future enhancements include:

- adaptive machine-learned cohort prediction,  
- multi-engine distributed anticipative rendering,  
- GPU-based anticipative processors,  
- per-Clip anticipative snapshots,  
- dynamic in-editor freeze visualisation,  
- multi-device anticipative clustering.

This architecture provides a strong foundation for all such extensions.
