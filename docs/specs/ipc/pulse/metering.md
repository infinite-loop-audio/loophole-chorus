# Pulse Metering Domain Specification

This document defines the Metering and Analysis domain of the Pulse project
protocol.  
Metering streams carry **high-rate, engine-originated** analysis data such as:

- audio meters (peak, RMS, LUFS),
- spectrum bins,
- stereo correlation,
- waveform previews,
- plugin-provided analyser output (if supported),
- timing and latency diagnostics.

Metering streams use a **dual-path architecture**:

- **Control-plane (Pulse ↔ Signal, Pulse ↔ Aura):**  
  Create/destroy streams, negotiate format, select metering modes, subscribe.
- **Data-plane (Signal ↔ Pulse):**  
  High-rate binary stream delivering analysis frames from Signal to Pulse.

Pulse aggregates high-rate metering data from Signal and forwards curated updates to Aura via standard JSON IPC events. Pulse may also receive **low-rate summaries** for monitoring and UI synchronisation.

---

## Contents

- [1. Overview](#1-overview)  
- [2. Metering Model](#2-metering-model)  
  - [2.1 Control-Plane vs Data-Plane](#21-control-plane-vs-data-plane)  
  - [2.2 Sources and Targets](#22-sources-and-targets)  
  - [2.3 Meter Types and Formats](#23-meter-types-and-formats)  
  - [2.4 Reliability and Rate](#24-reliability-and-rate)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Stream Lifecycle](#31-stream-lifecycle)  
  - [3.2 Subscription and Mode Control](#32-subscription-and-mode-control)  
  - [3.3 Data-Plane Negotiation](#33-data-plane-negotiation)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 Stream Lifecycle Events](#41-stream-lifecycle-events)  
  - [4.2 Mode and Subscription Events](#42-mode-and-subscription-events)  
  - [4.3 Data-Plane Negotiation Events](#43-data-plane-negotiation-events)  
  - [4.4 Summary Events](#44-summary-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Realtime and Cohort Considerations](#6-realtime-and-cohort-considerations)  
- [7. Integration with Plugin Analysis](#7-integration-with-plugin-analysis)  
- [8. Relationship to Gesture, Parameter and Node Domains](#8-relationship-to-gesture-parameter-and-node-domains)

---

## 1. Overview

Metering streams provide high-rate, realtime analysis of audio and DSP
state. They:

- give Aura the data it needs to render meters, spectrograms and other visual
  feedback,
- keep UI timing aligned with engine reality,
- support low-latency workflows where metering is used live (mixing, live
  monitoring),
- integrate plugin-provided analyser outputs,
- drive engine diagnostics.

Metering streams flow **Signal → Pulse → Aura**, with Pulse aggregating high-rate data from Signal and forwarding curated events to Aura.

They are generally session-scoped and ephemeral.

---

## 2. Metering Model

### 2.1 Control-Plane vs Data-Plane

#### **Control-plane (Pulse ↔ Signal, Pulse ↔ Aura)**
Manages:

- stream creation,
- format negotiation,
- enabling/disabling meter types,
- subscription state,
- routing analysis to the right UI panels,
- low-rate summaries for Pulse's internal monitoring.

#### **Data-plane (Signal ↔ Pulse)**

Carries high-rate data such as:

- peak/RMS frame values,
- FFT bins,
- loudness envelopes,
- multi-channel vectors (surround),
- correlation/goniometer data,
- analysis plugin output.

This data is:

- binary,
- loss-tolerant,
- timestamped or frame-indexed,
- non-JSON,
- rate-managed by Signal.

Pulse receives the high-rate stream from Signal, aggregates it, and forwards curated updates to Aura via standard JSON IPC events. Aura never establishes a direct connection to Signal.

---

### 2.2 Sources and Targets

Metering sources include:

- **Channels** (Track Channels, buses, FX returns, master),
- **Nodes** (AnalyzerNode outputs, PluginNode analysers if supported),
- **SendNode taps** (optional),
- **Hardware I/O** (input monitoring, output diagnostics),
- **Plugin-provided analysis paths** (CLAP extension or AU/VST vendor APIs).

A metering stream has:

- a `meterStreamId`,
- a `sourceId` (typically a `channelId` or `nodeId`),
- a `meterType` (peak/RMS/FFT/etc.),
- format metadata (frame size, bin count, units),
- a transport endpoint for the data-plane.

---

### 2.3 Meter Types and Formats

Common meter types include:

- **peak** (per-channel, instantaneous or short-window),
- **RMS** (short-window power),
- **LUFS / integrated loudness**,
- **spectrum** (FFT magnitude bins),
- **phase/correlation**,
- **goniometer vectors**,
- **waveform preview** (coarse resolution for editing),
- **latency/compensation diagnostics**,
- **plugin-specific analysis** (arbitrary binary payload or format-defined).

Format negotiation covers:

- sample rate of delivered frames,
- frame size,
- channel count,
- units (linear/DSP, dBFS, normalised),
- FFT window type and bin size (if applicable).

---

### 2.4 Reliability and Rate

Metering streams:

- **must** deliver the newest available frames quickly,
- **may** drop intermediate frames,
- **should** maintain local ordering,
- **must not** block Signal’s realtime thread,
- may apply **rate limiting** when the UI is not visible,
- must allow Aura to request changes to update frequency.

Pulse can request decimation or disable metering entirely for background channels.

---

## 3. Commands (Aura → Pulse)

### 3.1 Stream Lifecycle

**`meter.createStream`**  
Request creation of a metering stream.

Fields:

- `sourceId` (channel/node),
- `meterType`,
- optional format hints,
- optional rate hints.

Pulse returns:

- `meterStreamId`.

Signal prepares the backend and triggers negotiation events.

---

**`meter.destroyStream`**  
Terminate a metering stream.

Fields:

- `meterStreamId`.

Signal tears down the data-plane channel and stops producing analysis.

---

### 3.2 Subscription and Mode Control

**`meter.subscribe`**  
Enable delivery of metering data.

Fields:

- `meterStreamId`,
- optional `rateHint`.

**`meter.unsubscribe`**  
Pause or stop delivery.

This is useful for hidden UI panels or inactive tabs.

---

### 3.3 Data-Plane Negotiation

**`meter.negotiateDataChannel`**  
Request or reconfigure the data-plane channel.

Fields:

- `meterStreamId`,
- `protocol` (`"binary"`, `"shared-mem"`, `"pipe"`, `"webrtc"`),
- optional buffer parameters.

Pulse forwards the negotiation request to Signal and triggers events.

---

## 4. Events (Pulse → Aura)

### 4.1 Stream Lifecycle Events

**`meter.streamCreated`**  
Metering stream is ready.

**`meter.streamDestroyed`**

---

### 4.2 Mode and Subscription Events

**`meter.subscribed`**  
Acknowledges subscription.

**`meter.unsubscribed`**  
Acknowledges unsubscription.

### 4.3 Data-Plane Negotiation Events

**`meter.dataChannelReady`**  
Indicates that the Signal↔Pulse binary stream is established and Pulse is ready to forward metering data to Aura.

Includes:

- endpoint/URI/shm handle/port,
- meter format summary,
- refresh rate.

**`meter.dataChannelFailed`**

---

### 4.4 Summary Events

Pulse aggregates high-rate metering data from Signal and forwards **curated events** to Aura, including **low-frequency summaries** such as:

- peaks,
- RMS,
- LUFS,
- latest FFT slice for thumbnails,
- overload/warning flags,
- clipping indicators.

Event:

**`meter.summaryUpdated`**

Fields:

- `meterStreamId`,
- summary payload (meter-type dependent).

Aura uses this for UI refresh in low-rate contexts, or when the high-rate
data-plane is not active.

---

## 5. Snapshot Semantics

Metering streams are **not persisted** in project snapshots.

Snapshots do not include:

- `meterStreamId`s,
- stream configuration,
- UI subscription status.

However, **Channel metering modes** (e.g. pre-fader vs post-fader in the mixer)
may be stored as part of Channel metadata in other specs.

---

## 6. Realtime and Cohort Considerations

Metering interacts with cohorts as follows:

- live cohort nodes produce metering data directly,
- anticipative cohort nodes may produce:
  - coarse-resolution metering,
  - preview-only data,
  - or delayed metering if preview buffers exist.

Signal must ensure:

- metering never blocks audio processing,
- backpressure is handled by dropping frames,
- data is delivered opportunistically.

Pulse may:

- throttle or disable metering for unfocused Channels,
- request format/rate changes,
- track overload flags for diagnostics.

---

## 7. Integration with Plugin Analysis

If a plugin exposes analysis outputs:

- via CLAP’s analysis extensions,
- via custom vendor APIs,
- via side-band analysis ports,

then Pulse may expose these as metering streams:

- source = PluginNode,
- meterType = plugin-defined,
- format metadata defined by the plugin.

This allows uniform treatment of:

- built-in analysers,
- plugin analysers,
- Channel/Node-level analysis.

---

## 8. Relationship to Gesture, Parameter and Node Domains

Metering interacts with other domains as follows:

### **Gesture**  
Opposite flow direction.  
Both share negotiation patterns and dual-plane semantics.

### **Parameter**  
Metering data may trigger UI highlighting or warnings for specific parameters
(e.g. overload on a GainNode). No direct IPC dependency.

### **Automation**  
No direct dependency; metering is read-only analysis.

### **Node domain**  
Most metering sources are AnalyzerNodes or Channel endpoints.

### **Mixing model**  
Mixer faders/pans/sends are driven by parameter and gesture domains, and use
metering streams to render visual feedback.
