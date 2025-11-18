# Signal IPC – Media Domain

This document defines the **Media** IPC domain between Pulse and Signal.

The Media domain is responsible for:

- decoding media files (primarily audio, later video) into buffers usable by the
  Signal engine,
- providing streaming decoders for long-form material,
- generating analysis data (waveform overviews, spectral summaries, transient
  maps, loudness curves),
- reporting media-availability issues (missing files, slow/remote sources).

Pulse owns:

- the project’s concept of media items (IDs, locations, metadata, references
  from Clips and Lanes),
- storage layout decisions,
- cloud/offline handling at the UI / project level.

Signal owns:

- the mechanics of decoding and analysing media,
- the efficient integration of media into the processing graph.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Media domain:

- opens and decodes media referenced by Pulse,
- provides limited-lifetime decoded buffers (for e.g. one-shot Clips),
- provides streaming decoders for long files (background tracks, stems),
- performs analysis tasks:
  - waveform overview generation,
  - transient/beat maps,
  - loudness and peak envelopes,
  - spectral summaries (for visualisation or later spectral tools),
- reports problems:
  - file not found,
  - unsupported format,
  - decode errors,
  - slow/stalled reads (e.g. cloud hydration delays).

### 1.2 Non-Goals

The Media domain does **not**:

- decide which media to use where (that’s Pulse/Clips/Lanes),
- manage library browsing or tagging (Media Library architecture + Composer),
- implement recording sinks (capture is coordinated by Transport + Hardware +
  a Recording domain, not Media),
- perform full-blown spectral editing (that’s a future, higher-level feature),
- handle network/cloud synchronisation (Dropbox/iCloud/Composer), apart from
  reporting slow/unavailable reads.

---

## 2. Identity & Addressing

### 2.1 Media IDs

Pulse is responsible for assigning **Media IDs** and mapping them to physical
locations. Signal treats these IDs as opaque.

Example:

```json
{
  "mediaId": "media:pool:1234",
  "kind": "audio",
  "format": "wav"
}
```

Pulse must also provide a **concrete location** for Signal:

- absolute filesystem path, or
- platform-specific URI/token (e.g. macOS security-scoped bookmark), or
- a pseudo-path plus a handle if the OS requires special access semantics.

### 2.2 Sample Format & Rate

Signal decides the **decode format** suitable for its engine:

- typically decoded to:
  - 32-bit float,
  - engine’s effective sample rate,
  - interleaved or non-interleaved depending on internals.

Pulse does not dictate how the media is stored in memory; it only cares about:

- sample rate,
- length (in samples),
- channel count.

---

## 3. Message Flow Overview

Typical flows:

1. **On project load / Clip creation**
   - Pulse informs Signal that Clip X references Media Y.
   - Either:
     - Signal lazily decodes on first playback, or
     - Pulse explicitly calls `media.prepareDecode` / `media.requestDecode`.

2. **For long-form streaming**
   - Pulse requests a streaming decoder via `media.openStream`.
   - Signal returns a `streamId`.
   - The graph uses that `streamId` to pull audio in real time.

3. **For analysis**
   - Pulse requests `media.analyseWaveform`, etc.
   - Signal schedules analysis and returns progress + completion events.

4. **On error**
   - Signal emits `media.error`, `media.unavailable`, or `media.slowRead`.
   - Pulse reacts by:
     - marking Clips as offline,
     - requesting re-tries later,
     - showing appropriate UI.

---

## 4. Commands (Pulse → Signal)

### 4.1 `media.prepare`

Hint to Signal that a given media item will be needed soon.

**Request**

```json
{
  "command": "media.prepare",
  "mediaId": "media:pool:1234",
  "location": {
    "path": "/Users/user/Projects/Loophole/Media/snare_01.wav"
  },
  "options": {
    "usage": "clip",           // "clip" | "preview" | "background"
    "priority": "normal"       // "low" | "normal" | "high"
  }
}
```

Semantics:

- Signal should:
  - validate existence,
  - open underlying resource,
  - optionally begin background decoding or caching.
- No strict response beyond generic OK/error; actual readiness is signalled by
  analysis/availability events.

---

### 4.2 `media.release`

Inform Signal that media is no longer actively needed.

**Request**

```json
{
  "command": "media.release",
  "mediaId": "media:pool:1234"
}
```

Semantics:

- A hint, not a hard guarantee:
  - Signal is free to keep caches if memory allows,
  - but may use this to prioritise freeing resources.

---

### 4.3 `media.openStream`

Open a streaming decoder for a media item.

**Request**

```json
{
  "command": "media.openStream",
  "mediaId": "media:pool:long-ambient",
  "location": {
    "path": "/path/to/long_ambient.wav"
  },
  "options": {
    "startSamples": 0,
    "endSamples": null,
    "loop": false
  }
}
```

**Response**

```json
{
  "replyTo": "media.openStream",
  "streamId": "stream:media:long-ambient",
  "mediaInfo": {
    "sampleRate": 48000,
    "channels": 2,
    "lengthSamples": 123456789
  }
}
```

Semantics:

- Signal creates a streaming decoder bound to `streamId`.
- Graph Nodes (e.g. SamplerNode or MediaNode) can then attach to `streamId` and
  pull audio as needed.
- Pulse normally does not interact with `streamId` directly beyond creation and
  closing; it’s an engine-level handle.

---

### 4.4 `media.closeStream`

Close a previously opened media stream.

**Request**

```json
{
  "command": "media.closeStream",
  "streamId": "stream:media:long-ambient"
}
```

Semantics:

- Frees resources for that stream.
- The engine should handle any currently-playing Nodes gracefully (e.g. fade to
  silence, or fail safe).

---

### 4.5 `media.requestInfo`

Request basic information about a media file.

**Request**

```json
{
  "command": "media.requestInfo",
  "mediaId": "media:pool:1234",
  "location": {
    "path": "/path/to/snare_01.wav"
  }
}
```

**Response**

```json
{
  "replyTo": "media.requestInfo",
  "mediaId": "media:pool:1234",
  "info": {
    "sampleRate": 44100,
    "channels": 1,
    "lengthSamples": 22050,
    "bitDepth": 24,
    "format": "wav"
  }
}
```

Semantics:

- Used by Pulse when it needs to confirm length/sample rate, or for compatibility
  checks when importing media.

---

### 4.6 `media.analyseWaveform`

Request generation of a multi-resolution waveform overview.

**Request**

```json
{
  "command": "media.analyseWaveform",
  "mediaId": "media:pool:1234",
  "location": {
    "path": "/path/to/snare_01.wav"
  },
  "options": {
    "resolutions": [256, 1024, 4096]
  }
}
```

**Response**

```json
{
  "replyTo": "media.analyseWaveform",
  "jobId": "mediaJob:waveform:1234"
}
```

Semantics:

- Signal schedules an analysis job with ID `jobId`.
- Results are delivered asynchronously via `media.waveformReady`.

---

### 4.7 `media.analyseTransientMap`

Request transient/beat detection.

**Request**

```json
{
  "command": "media.analyseTransientMap",
  "mediaId": "media:pool:drumloop",
  "location": {
    "path": "/path/to/drumloop.wav"
  },
  "options": {
    "sensitivity": 0.7
  }
}
```

**Response**

```json
{
  "replyTo": "media.analyseTransientMap",
  "jobId": "mediaJob:transients:drumloop"
}
```

Semantics:

- Results delivered via `media.transientMapReady`.
- Pulse can feed this into:
  - Clip slicing,
  - groove extraction,
  - time-stretch warp markers.

---

### 4.8 `media.analyseLoudness`

Request loudness/level envelope analysis.

**Request**

```json
{
  "command": "media.analyseLoudness",
  "mediaId": "media:pool:1234",
  "location": {
    "path": "/path/to/snare_01.wav"
  },
  "options": {
    "windowSizeSamples": 2048,
    "hopSizeSamples": 1024
  }
}
```

**Response**

```json
{
  "replyTo": "media.analyseLoudness",
  "jobId": "mediaJob:loudness:1234"
}
```

---

### 4.9 `media.analyseSpectralSummary`

Request a coarse spectral summary suitable for:

- media clustering,
- visualisation (spectral thumbnails),
- basic tonal character analysis.

**Request**

```json
{
  "command": "media.analyseSpectralSummary",
  "mediaId": "media:pool:texture",
  "location": {
    "path": "/path/to/texture.wav"
  },
  "options": {
    "bands": 32
  }
}
```

**Response**

```json
{
  "replyTo": "media.analyseSpectralSummary",
  "jobId": "mediaJob:spectral:texture"
}
```

Semantics:

- This is a coarse summary, not full-resolution spectral analysis.
- May be used by Composer and Aura to drive cluster visualisations (XO-style
  clouds, hypergrid browsers, etc.).

---

## 5. Events (Signal → Pulse)

### 5.1 `media.available`

Media has been successfully opened and is ready.

**Payload**

```json
{
  "event": "media.available",
  "mediaId": "media:pool:1234",
  "info": {
    "sampleRate": 44100,
    "channels": 1,
    "lengthSamples": 22050,
    "format": "wav"
  }
}
```

Semantics:

- May be emitted in response to `media.prepare` or as part of analysis tasks.
- Pulse can use this to flip Clips from “pending” to “ready”.

---

### 5.2 `media.unavailable`

Media could not be opened or decoded.

**Payload**

```json
{
  "event": "media.unavailable",
  "mediaId": "media:pool:1234",
  "reason": "notFound", // "notFound" | "unsupportedFormat" | "decodeError" | "permissionDenied" | "unknown"
  "message": "File not found at given path.",
  "details": {
    "path": "/path/to/snare_01.wav"
  }
}
```

Pulse must treat this as authoritative and:

- mark media as offline for this session,
- optionally prompt the user to locate or replace the file.

---

### 5.3 `media.slowRead`

Media access is slow or blocked (e.g. cloud storage hydrating, external drive
spinning up, network mount lag).

**Payload**

```json
{
  "event": "media.slowRead",
  "mediaId": "media:pool:largeLoop",
  "message": "Read operation is taking unusually long.",
  "details": {
    "path": "/path/to/largeLoop.wav",
    "approxDelayMs": 1200
  }
}
```

Semantics:

- Pulse should:
  - avoid blocking the UI,
  - show non-blocking indicators if needed,
  - avoid repeatedly hammering the same path.

This supports the requirement that missing or slow media **must not** make the
entire app feel frozen.

---

### 5.4 `media.waveformReady`

Waveform overview data is ready.

**Payload**

```json
{
  "event": "media.waveformReady",
  "jobId": "mediaJob:waveform:1234",
  "mediaId": "media:pool:1234",
  "resolutions": [
    {
      "windowSize": 256,
      "minMaxPairs": [ /* compressed data */ ]
    },
    {
      "windowSize": 1024,
      "minMaxPairs": [ /* compressed data */ ]
    }
  ]
}
```

The exact encoding of `minMaxPairs` (e.g. per-window min/max in normalised
float, potentially delta-encoded) is an implementation detail; the IPC schema
just treats it as an opaque array.

---

### 5.5 `media.transientMapReady`

Transient / beat map is ready.

**Payload**

```json
{
  "event": "media.transientMapReady",
  "jobId": "mediaJob:transients:drumloop",
  "mediaId": "media:pool:drumloop",
  "transients": [
    {
      "positionSamples": 1024,
      "strength": 0.9
    },
    {
      "positionSamples": 4096,
      "strength": 0.7
    }
  ]
}
```

Pulse uses this to:

- suggest slice markers,
- derive groove templates,
- inform time-stretch warp markers.

---

### 5.6 `media.loudnessReady`

Loudness / peak envelope is ready.

**Payload**

```json
{
  "event": "media.loudnessReady",
  "jobId": "mediaJob:loudness:1234",
  "mediaId": "media:pool:1234",
  "envelope": {
    "windowSizeSamples": 2048,
    "hopSizeSamples": 1024,
    "values": [ /* loudness values, implementation-defined units */ ]
  }
}
```

---

### 5.7 `media.spectralSummaryReady`

Coarse spectral summary is ready.

**Payload**

```json
{
  "event": "media.spectralSummaryReady",
  "jobId": "mediaJob:spectral:texture",
  "mediaId": "media:pool:texture",
  "bands": 32,
  "values": [ /* e.g. average energy per band, normalised */ ]
}
```

Composer and the Media browser can use this for clustering and search.

---

## 6. Error Handling

Media commands use the standard error envelope when they fail *synchronously*:

- invalid media ID,
- missing/invalid location,
- unsupported options.

Asynchronous failures (during decode or analysis) are surfaced via:

- `media.unavailable`,
- `media.slowRead`,
- job-specific failures (not yet enumerated, but can be added as `media.jobFailed`).

Example synchronous error:

```json
{
  "error": {
    "code": "invalidLocation",
    "message": "No usable path or handle provided for media.",
    "domain": "media",
    "details": {
      "mediaId": "media:pool:1234"
    }
  }
}
```

---

## 7. Relationship to Other Signal Domains

- **signal.graph**  
  Graph Nodes that play media (SamplerNode, MediaNode, etc.) depend on Media
  to supply decoded buffers or streams.

- **signal.engine**  
  Decoding and analysis may be constrained by overall engine configuration and
  threading model, but are generally non-real-time tasks.

- **signal.transport**  
  Streaming decoders must follow the transport position; pre-roll and loop
  regions may cause additional buffering demands.

- **signal.diagnostics**  
  Media-related performance issues (slow reads, CPU load spikes during decode)
  can be surfaced in Diagnostics.

---

## 8. Extensibility

Future extensions might include:

- direct integration with Composer for cloud-hosted content,
- on-the-fly format conversion and normalisation,
- partial spectral analysis per segment,
- video track media support (synchronised to audio transport),
- audio-to-MIDI and key detection hooks.

Any such features should build on the basic model of Media as a provider of
decoded buffers, streaming data, and analysis results, without overloading the
core real-time audio engine.
