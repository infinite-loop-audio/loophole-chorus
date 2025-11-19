# Diagnostics & Performance Architecture

This document defines the **diagnostics, performance analysis and engine
introspection architecture** for Loophole.

It covers:

- realtime engine monitoring  
- anticipative rendering state  
- per-track and per-node performance metrics  
- plugin stability and crash isolation  
- profiling and benchmarking  
- parameter tracing / modulation tracing  
- ARA & pipeline diagnostics  
- hardware I/O inspection  
- task logs and structured events  
- Composer telemetry interactions  
- UX hooks for diagnostics view  

Diagnostics is fundamental to Loophole’s reliability and developer-facing
tooling. It also powers user-facing insights into performance bottlenecks,
plugin stability, graph behaviour and workflow optimisation.

It builds on:

- `06-processing-cohorts-and-anticipative-rendering.md`
- `11-node-graph.md`
- `12-mixer-and-channel-architecture.md`
- `16-editing-and-nondestructive-layers.md`
- `20-rendering-and-offline-processing.md`
- `21-control-surfaces-and-assistive-hardware.md`
- Signal & Pulse architecture docs

---

## 1. Goals

Diagnostics must provide:

1. **Transparent visibility** into every part of the audio engine:
   - DSP graph performance,
   - automation/modulation load,
   - ghost render status,
   - disk I/O,
   - plugin behaviour.

2. **Crash resilience**:
   - plugin crashes must not crash Signal,
   - Signal crashes must not crash Pulse,
   - Pulse crashes must not lose project state.

3. **Real-time correctness**:
   - stable low-latency playback even under high CPU load,
   - clear indication when real-time safety is threatened.

4. **Developer-friendly debugging**:
   - per-node timing,
   - block-by-block execution logs (for debug builds),
   - rendering graphs for inspection.

5. **User-friendly performance insights**:
   - clear, non-technical indicators of heavy tracks or plugins,
   - warnings before dropouts occur,
   - suggestions powered by Composer if patterns are detected.

---

## 2. Diagnostics Subsystem Overview

Diagnostics consists of four main layers:

1. **Signal Metrics Engine** (inside Signal)
2. **Pulse Aggregator & Event Model** (inside Pulse)
3. **Composer Telemetry Layer**
4. **Aura Diagnostic UX**

These form a flow:

```
Signal → Pulse → (Composer) → Aura
```

Where each layer adds interpretation, aggregation or visual presentation.

---

## 3. Signal Metrics Engine

Signal captures raw performance metrics with minimal overhead.

### 3.1 Per-Node Metrics

Collected per processing block:

```
NodeMetrics {
  nodeId;
  lastBlockDurationMicros;
  avgBlockDurationMicros;
  peakBlockDurationMicros;
  xrunsCount;
  oversampleFactor?;
  bufferSize;
  processingCohort;     // realtime or anticipative
}
```

### 3.2 Graph Metrics

```
GraphMetrics {
  realtimeCohortLoad;
  anticipativeCohortLoad;
  totalLoad;
  bufferUnderruns;
  bufferOverruns;
  pluginCrashes;
}
```

### 3.3 Disk I/O Metrics

For streaming large files:

```
DiskIOMetrics {
  bytesRead;
  cacheHits;
  cacheMisses;
  latencyMicros;
}
```

### 3.4 Hardware Metrics

From device engines:

- MIDI/HID/OSC packet rates  
- dropped packets  
- transport latency  

### 3.5 Gesture/Value-Stream Metrics

For high-resolution input streams:

- decimation rate  
- smoothing overhead  
- session duration  
- dropped packets  

### 3.6 ARA Metrics

For ARA nodes:

- region analysis duration  
- render overhead  
- memory usage  

Signal sends metrics as periodic updates or on-demand snapshots.

---

## 4. Pulse Aggregator & Event Model

Pulse receives raw metrics and converts them into structured diagnostic events.

### 4.1 Event Types

```
DiagnosticEvent {
  eventId;
  timestamp;
  severity: info | warn | error | critical;
  source: signal | pulse | plugin | device | renderer;
  message;
  metadata;
}
```

Common events:

- plugin crash or timeout  
- buffer underrun / overrun  
- node overload (block time > buffer)  
- anticipative engine rebalancing  
- ghost render pending/failed  
- ARA reanalysis triggered or stalled  
- device disconnect  
- device jitter above threshold  
- incoming data storms (e.g. faulty MIDI device)  

### 4.2 Aggregated Metrics

Pulse aggregates metrics by:

- track  
- node  
- cohort  
- device  
- project-wide totals  

And stores short-term history for graphs.

### 4.3 Diagnostic Log

Pulse maintains a structured rolling log:

```
diagnostics.log.jsonl
```

With JSON lines for each event.

User can choose to:
- export logs,
- send anonymised logs to Composer,
- attach logs to bug reports.

---

## 5. Plugin Isolation & Stability Model

Pulse treats plugins and heavy ARA operations as fragile by default.

### 5.1 Crash Isolation

- Signal isolates plugin execution per node or per block (host-configurable).
- On plugin crash:
  - isolate the faulty node,
  - replace with a bypass/dummy node,
  - notify Pulse with severity “error”.

Pulse then:
- marks plugin/node as unstable,
- optionally disables anticipative rendering for that node,
- records plugin crash metrics in Composer.

### 5.2 Timeout & Stall Detection

If a node takes too long:
- Signal cuts it off,
- fallback block is substituted,
- Pulse logs event and reports to user.

### 5.3 Sandbox Levels

Loophole can support multiple sandbox levels:

- **Strict**: each plugin isolated; restartable individually.
- **Moderate**: plugin groups isolated.
- **Relaxed**: single-process plugin execution for low-latency configs.

---

## 6. Anticipative Engine Diagnostics

From `06-processing-cohorts-and-anticipative-rendering.md`.

Pulse must surface:

- anticipative progress / buffer fill %
- nodes migrated between cohorts
- heavy nodes that trigger anticipative pre-render
- nodes marked “non-deterministic” by user
- auto-detected non-deterministic nodes (from Composer telemetry)
- freeze candidates (Pulse can suggest tracks to freeze)

UI indicators should show:

- cohort membership of each node,
- whether realtime-safe operation is in danger,
- when anticipative pre-buffering is underway.

---

## 7. Clip Pipeline & Ghost Render Diagnostics

Pulse tracks clip rendering pipelines:

- which clips need ghost render,
- ghost render progress (%) for each,
- render failures or stalls,
- ARA reanalysis scheduling,
- lineage/debug info for derived assets.

Aura surfaces:
- “Clip is being rendered…”
- “Ghost render updated.”
- “Clip render stalled – see Diagnostics.”

---

## 8. Node Graph Diagnostics

Per NodeGraph (track-level):

- execution order  
- CPU cost per node  
- latency per node  
- oversampling or zero-latency modes  
- graph cycles detection (Pulse enforces acyclic)  
- implicit sends/returns resolved properly  
- Cacheable vs non-cacheable nodes  

Aura can provide:
- a developer “graph view”,
- a simplified user “heat map” (nodes coloured by CPU load).

---

## 9. Automation & Modulation Diagnostics

Pulse tracks:

- automation envelope density (points per beat)  
- modulation rate & depth  
- combined modulation CPU cost  
- automation jitter in sample domain  
- gesture sessions reconciling with automation  

Aura may show:
- highlight overloaded modulation chains,
- warn about aliasing when modulating very fast parameters.

---

## 10. Hardware Diagnostics

Pulse stores:

- per-device jitter (HID/MIDI),
- dropped packets,
- out-of-range or malfunctioning controls,
- device heartbeat failures,
- mode handshakes (sysex) that didn’t complete.

Aura shows:
- device health indicators (“OK”, “Degraded”, “Unresponsive”),
- warnings for noisy or broken controls,
- colour-coded performance meters for control message rate.

Composer may incorporate this into global telemetry:
- “This device model typically shows jitter on encoder 3; recommend filtering mode X.”

---

## 11. Disk & Media Diagnostics

Pulse tracks:

- disk read throughput,
- disk I/O latency,
- media cache hit/miss statistics,
- ghost-render storage usage,
- project size and archive status.

Aura may display:
- warnings when large media is being streamed from slow drives,
- progress on clip consolidation or media archiving.

---

## 12. Rendering Diagnostics

During offline rendering:

Pulse displays:
- current phase (analysis, graph build, audio render, finalisation),
- per-track render progress,
- per-node render times,
- warnings (oversampling, ARA stalls, node crashes),
- final summary of issues encountered.

Offline rendering log is stored for later review and Composer telemetry.

---

## 13. Diagnostics UX (Aura)

Aura provides:

### 13.1 Diagnostics Panel

A dedicated tab showing:
- CPU graphs (overall, realtime, anticipative),
- per-track CPU meters,
- device health overview,
- graph heatmap,
- clip ghost-render backlog,
- plugin stability chart.

### 13.2 Event Log View

- structured list of DiagnosticEvents,
- filters (severity, source, track, plugin),
- click-to-focus (e.g. jump to problem node),
- export to JSONL.

### 13.3 Node Graph Visualiser

Developer-friendly view:
- node graph layout,
- CPU, latency, cohort,
- connections & signal paths,
- red/yellow colouring on overloaded nodes.

### 13.4 Hardware Test Mode

Exercises:
- faders, encoders, grids, pads,
- LED feedback,
- jitter tests,
- MIDI/HID packet monitor.

### 13.5 Timeline View Integration

Tracks can show subtle badges:
- “peak CPU exceeded 60%”,
- “clip pipeline pending render”,
- “plugin instability detected”.

---

## 14. Composer Telemetry

Pulse may send anonymised telemetry to Composer:

- plugin crash signatures,
- device jitter profiles,
- performance metrics correlated with plugin types or node chains,
- heavy track patterns that Composer can use for suggestions.

Composer returns improvement suggestions:
- “Freeze these tracks for better realtime performance”,
- “Plugin X is known to be unstable; enable sandbox mode.”
- “Node Y has unusually heavy modulation load; consider thinning automation.”

Users may opt-in/out.

---

## 15. Versioning & Persistence

Diagnostics data is not part of project state but may persist:

- per-project:
  - recent diagnostics events,
  - stability history,
  - “known unstable nodes”.

- per-machine:
  - device jitter fingerprints,
  - plugin instability patterns.

Pulse stores only short histories by default.

---

## 16. Summary

The Diagnostics & Performance subsystem provides:

- deep visibility into every layer of the engine,
- nondestructive crash isolation & recovery,
- performance feedback for users and developers,
- monitoring of automation, modulation, rendering, hardware and DSP,
- strong integration with Composer,
- robust UX for investigating issues,
- a foundation for self-tuning systems and intelligent suggestions.

This architecture is central to Loophole’s promise of reliability, low-latency
performance and transparency.
