# Audio Streaming & Caching Architecture

This document defines the architecture for **audio file streaming, caching,
prefetching, and disk I/O** within Loophole.

It complements and extends:

- `13-media-architecture.md`
- `09-clips.md`
- `16-editing-and-nondestructive-layers.md`
- `20-rendering-and-offline-processing.md`
- `23-diagnostics-and-performance.md`
- Signal architecture (disk I/O responsibilities)
- Pulse architecture (metadata, clip pipelines, caching policies)

Audio streaming sits at the junction of:
- **Signal** → performs the actual file reads, decoding and buffering  
- **Pulse** → manages asset metadata, caching policies and anticipative scheduling  
- **Aura** → displays status, thumbnails, waveform previews  
- **Composer** → may provide heuristics for caching and performance prediction  

The goal is a modern media engine supporting extremely large projects,
spanning thousands of audio clips, in real-world production workloads.

---

# 1. Goals

Loophole’s audio streaming model must:

1. **Be reliable:**
   - No stalls, no blocking, no UI freezes, no “media offline” surprises.
   - Graceful handling of missing drives, cloud-delayed files, or Dropbox-style placeholders.

2. **Be scalable:**
   - Support huge sessions.
   - Only stream what is needed.
   - Avoid overloading the disk under heavy playback.

3. **Be anticipative-aware:**
   - When anticipative processing is active, the streaming subsystem must also pre-read required audio.

4. **Be cache-friendly:**
   - Smart cache windows around playhead.
   - Per-clip prefetching.
   - Predictive caching for looping and launcher playback.

5. **Be coherent with destructive and non-destructive editing:**
   - Edits that generate ghost renders automatically update cache.
   - Pipelines that modify audio use cached pre-render buffers.

6. **Provide fast access to waveform preview data:**
   - Waveform caches and resolution-aware previews.
   - Zero latency browsing in the media browser.

---

# 2. Audio Asset Model

Pulse owns the metadata describing each audio file.

```
AudioAsset {
  assetId;
  path;
  type: sample | stem | recording | render | proxy | transient;
  sampleRate;
  channels;
  bitDepth;
  lengthSamples;
  peakCache?;
  waveformCache?;
  proxyStatus;
  hash?;               // used for fingerprinting & file integrity
  offlineStatus;       // present | missing | cloud | pending
}
```

### 2.1 Proxy Assets

A proxy is:
- lower sample rate or lower bit depth,
- optionally mono downmix for preview,
- used for heavy assets that don’t require full fidelity during editing.

Proxies follow the same ID format:
```
ProxyAsset {
  parentAssetId;
  quality;
  path;
}
```

Signal decides whether to use full-resolution or proxy based on:
- Playback resolution,
- User preference,
- System load.

---

# 3. Audio Streaming Overview

Signal handles all audio streaming.  
Pulse never touches raw sample data.

Flow:

```
Pulse → Signal → Disk → Signal Cache → DSP Node
```

### 3.1 Two-Tier Streaming Model

Signal maintains **two levels of audio data**:

1. **Disk Stream Buffers** (low-level, continuous)
2. **Clip-Window Buffers** (high-level, clip-aligned, anticipative-aware)

This dual model allows:
- efficient reuse of buffers,
- anticipative reads aligned with cohort scheduling,
- low-latency scrubbing and preview.

---

# 4. Disk Streaming Layer (Signal)

This is the engine’s foundation.

## 4.1 Streaming Worker Threads

Signal creates a small controlled pool:

```
DiskIOThreadPool = {
  N threads, priority low, preemptible
}
```

Responsibilities:
- Asynchronous disk reads,
- Buffered read-ahead,
- Decode operations (if the asset is compressed),
- Feeding Stream Buffers.

## 4.2 Stream Buffers

Each active clip or region gets a **StreamHandle**:

```
StreamHandle {
  handleId;
  assetId;
  positionSamples;
  buffer[];            // small ring buffer
  state: filling | ready | underrun
  lastReadTime;
}
```

These buffers keep a “just enough” window around read-ahead.

## 4.3 Underrun Protection

Signal implements:
- dynamic buffer expansion under load,
- overruns reported as diagnostics,
- slow-disk protection (fallback to proxy mode).

---

# 5. Clip-Window Buffers (Pulse + Signal)

Pulse uses clip and timeline context to give Signal **ClipWindow requests**:

```
ClipWindow {
  clipId;
  startSample;
  endSample;
  isLooping;
  playRate;
  cohort;                  // realtime or anticipative
}
```

Signal then:
- populates a larger clip-local buffer,
- transitions the clip between streaming and cached modes.

These windows are:
- essential for anticipative rendering,
- updated when user performs timeline jumps or edits,
- responsible for ensuring seamless loop playback.

---

# 6. Anticipative Rendering Integration

From `06-processing-cohorts-and-anticipative-rendering.md`.

## 6.1 Cohort-Aware Prefetching

When anticipative processing prepares future blocks:
- Pulse sends **ClipWindow requests** for future sections of the timeline,
- Signal prefetches audio data accordingly.

## 6.2 Prefetch Priority

Priority logic:

1. realtime cohort > anticipative cohort  
2. near-future > far-future  
3. small assets > long assets  
4. cached segments > uncached segments  

Pulse assists by providing:
- the exact future timeline segments required,
- which clips are expected to play,
- which sections need ghost renders.

---

# 7. Caching Model

Signal manages caches at three levels:

## 7.1 Hot Buffer Cache (per-stream)
- A few hundred milliseconds around playhead.
- Always in RAM.
- Used for immediate playback and scrubbing.

## 7.2 Warm Clip Cache
- Clip-level cache matching ClipWindow.
- Used in anticipative rendering.
- Potentially seconds-long.

## 7.3 Cold Global Asset Cache
- Waveforms and peak data stored on disk.
- Asset metadata loaded lazily on project open.

---

# 8. Waveform & Peak Cache

Waveform and peak data is stored as:

```
PeakCache {
  assetId;
  resolution;      // per-pixel or per-block
  peaks[];   
}
```

Waveform previews:
- never decoded on the fly during playback,
- are precomputed when assets are imported,
- regenerate only when destructive edits update the underlying asset.

---

# 9. Handling Missing or Cloud-Based Files

Pulse marks assets as:

- **present** – file is accessible  
- **cloud** – cloud placeholder, must be downloaded  
- **missing** – cannot be found  
- **pending** – download in progress  

Signal must:
- not block if file is unavailable,
- stream proxy or silence depending on asset type,
- notify Pulse of availability changes,
- retry when the file appears.

Aura:
- shows clear “cloud” or “offline” icons,
- does not spam the user with errors.

This resolves long-standing frustrations with Dropbox/OneDrive/Google Drive workflows.

---

# 10. Clip Pipelines & Streaming

Clip pipelines (from `16-editing-and-nondestructive-layers.md`) must integrate seamlessly.

## 10.1 Pre-Rendered Clip Buffers

When a clip pipeline produces a new intermediate render:
- Pulse assigns the render to a new `AudioAsset`,
- Signal replaces the streaming source handle,
- underlying disk I/O adapts automatically.

## 10.2 ARA Integration

ARA-based processing:
- behaves as another pipeline stage,
- may request additional streaming from Signal,
- produces updated media assets as needed.

---

# 11. Audio Format Support

Signal must support:

### 11.1 Uncompressed Formats
- WAV
- AIFF
- BWF (with timecode support)

### 11.2 Compressed Formats (Decode-on-fetch)
- FLAC (lossless)
- MP3 (decode for preview, rarely used for production)
- OGG
- AAC (platform-dependent)

### 11.3 Multi-channel
Up to:
- 7.1 for basic use,
- object-based extensions in future routing models.

Decode pipelines must be off realtime threads.

---

# 12. Diagnostics

Related to `23-diagnostics-and-performance.md`:

Signal reports:
- read time per block,
- cache misses,
- decode times,
- underruns,
- file availability changes.

Pulse aggregates:
- track impact,
- session-wide disk I/O,
- warnings if disk cannot sustain playback.

---

# 13. IPC Responsibilities

## Pulse → Signal
- `asset.register`
- `asset.updateStatus`
- `clipWindow.request`
- `stream.start` / `stream.stop`
- `render.requestClipGhost`

## Signal → Pulse
- `stream.underrun`
- `asset.statusChanged`
- `stream.prefetchComplete`
- `decode.error`

## Pulse ↔ Aura
- `asset.previewWaveform`
- `asset.previewPeaks`
- `clip.pipelineUpdated`

---

# 14. Summary

Loophole’s audio streaming and caching architecture:

- separates disk streaming and clip windows for efficient anticipative behaviour,
- guarantees stable, non-blocking media access,
- integrates with destructive and non-destructive clip pipelines,
- handles large sessions gracefully,
- supports proxy workflows,
- is resilient to missing/cloud assets,
- feeds into diagnostics and anticipative scheduling,
- ensures seamless scrubbing and playback.

This system provides modern DAW media reliability without relying on legacy behaviour or brittle disk I/O assumptions.
