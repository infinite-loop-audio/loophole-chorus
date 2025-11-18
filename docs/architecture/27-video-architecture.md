# Video Architecture

This document defines how Loophole supports **video playback, editing, synchronisation, rendering and analysis** within the hybrid audio–media engine.

Loophole is not intended to be a full NLE (non-linear video editor), but it must offer extremely strong **music-to-picture** capabilities, and provide a foundation for future expansion into richer video workflows.

This architecture integrates with:

- `08-tracks-lanes-and-roles.md`
- `09-clips.md`
- `10-parameters.md`
- `11-node-graph.md`
- `12-mixer-and-channel-architecture.md`
- `14-timebase-tempo-and-groove.md`
- `15-advanced-clips.md`
- `16-editing-and-nondestructive-layers.md`
- `20-rendering-and-offline-processing.md`
- `21-control-surfaces-and-assistive-hardware.md`
- `23-diagnostics-and-performance.md`  
- Pulse + Signal + Aura architecture docs  
- IPC event and command schemes  

Loophole video is deliberately minimal at first, but structurally extensible to become extremely powerful.

---

# 1. Goals

Loophole’s video system must:

1. **Synchronise audio & video with sample accuracy**, regardless of buffering strategy.
2. Support **multiple video lanes** on tracks (the same way audio lanes work).
3. Provide a **reference-grade preview**, not a full NLE timeline.
4. Allow **basic trim, slip, cut, stretch, and fade** operations.
5. Provide **timecode-accurate export** for scoring-to-picture workflows.
6. Support **proxy media**, **pre-renders**, and **low-latency decode pipelines**.
7. Integrate with the audio anticipative/rendering engine.
8. Support **ARA-like analysis** for frame-aware plug-ins in the future.
9. Keep rendering safe: video must **never** interfere with audio stability.
10. Allow for future advanced capabilities:  
   - video effects nodes  
   - colour grading  
   - video-layer compositing  
   - video-based modulation sources  
   - multi-camera assets  

---

# 2. Core Video Concepts

## 2.1 Video Assets

A video file imported into Loophole becomes a **VideoAsset**:

```
VideoAsset {
  assetId;
  path;
  metadata: {
    container;
    codec;
    fps;
    frameDuration;     // nanoseconds
    resolution;
    colourSpace;
    audioStreams[];
    duration;
  };
  proxyStatus;
  waveformCache?;
  frameCache?;
}
```

Pulse owns all metadata and proxy status.  
Signal handles decoding.

---

## 2.2 Video Lanes

Video Lanes are a **lane type** on Tracks:

```
Lane(type: "video") {
  clips: VideoClip[]
}
```

Video lanes behave exactly like audio lanes in the arrangement view:

- clips can overlap,
- clips can be layered (topmost wins),
- automation can be applied to video parameters (opacity, scale, etc.),
- NodeGraph generates a corresponding **VideoNode** pipeline.

---

## 2.3 Video Clips

A video clip contains:

```
VideoClip {
  clipId;
  assetId;
  laneId;
  inPoint;         // source media start
  outPoint;        // source media end
  startTime;       // where it sits in arrangement
  offset;          // slip position
  playbackRate;
  opacity;
  transforms[];
  pipeline: ClipPipeline[];
}
```

Clip pipelines mirror audio pipelines and support non-destructive processing chains.

---

## 2.4 Video Node Graph

Tracks with video lanes create a **VideoNodeGraph**, parallel to audio:

```
VideoNode {
  nodeId;
  type;                    // decoder, transformer, compositor, colour, overlay
  params;
  inputs[];
  outputs[];
}
```

VideoNodes integrate with anticipative rendering:

- video frames may be pre-decoded,
- pre-buffering similar to ghost renders,
- cache invalidation when edits occur.

---

# 3. Playback Architecture (Signal)

## 3.1 Principles

Video decoding must **never block** the realtime audio thread.

Therefore:

- all decode work happens on worker threads,
- video playback is always framed around **Pulse-managed timebase**,
- Signal’s video layer follows the transport state from Pulse,
- anticipative rendering can decode frames ahead where appropriate.

## 3.2 Decode Flow

```
Pulse → Signal (request video frame at timestamp X)
Signal:
  - checks frame cache
  - if uncached → decode in worker thread
  - returns frame handle to Pulse/Aura
Aura:
  - renders preview
```

Pulse **never** touches video frames directly—only timestamp handles and metadata.

## 3.3 Frame Cache

Signal manages:

```
FrameCache {
  frameId;
  timestamp;
  pixelDataPtr;       // GPU uploadable or CPU memory block
  resolution;
  colourSpace;
  refCount;
}
```

Frame eviction rules:
- LRU within memory budget,
- forced flush on asset change,
- anticipative buffer can override LRU temporarily.

---

# 4. Proxy System

Large video files require proxies:

```
ProxyStatus = none | generating | ready | failed
```

Pulse coordinates:
- proxy requests,
- cache lifetime,
- re-proxy after editing operations,
- storing proxy metadata in project file.

Signal:
- generates proxies using external codecs (ffmpeg-like integration),
- keeps GPU-friendly formats for preview.

Aura:
- shows proxy status badges,
- indicates when preview is proxy / full-res.

---

# 5. Editing Capabilities

Loophole initially supports:

### 5.1 Basic Edits
- trim in/out  
- slip  
- stretch (rate change)  
- cut / join  
- fade in/out (opacity)  

### 5.2 Transform Operations
- position  
- scale  
- rotation  
- anchor/pivot  
- simple cropping  

### 5.3 Clip Pipelines
Like audio pipelines, including:
- colour adjustment nodes  
- overlays / text (future)  
- LUTs (future)  
- frame-analysis nodes (future)

### 5.4 Multilayer Playback
Video Lanes stacked create implicit compositing:
- top = highest priority  
- bottom = background  

In the future this can be expanded into explicit compositing nodes.

---

# 6. Rendering Architecture

## 6.1 Offline Render

Pulse orchestrates:

1. build unified timeline model  
2. request video frames from Signal  
3. combine audio and video  
4. final mux into output container  

Signal:
- renders video frames in correct order,  
- can use GPU acceleration for scaling, colour, transforms.

## 6.2 Synchronisation

Everything syncs to **Pulse timeline**:
- video timestamps,
- frame boundaries,
- automation,
- audio buffer boundaries.

## 6.3 Variable Framerate Support

Loophole uses **timestamp-driven** rendering rather than frame index driven, ensuring:

- correct sync with VFR footage,
- predictable automation alignment.

---

# 7. Video Automation & Parameters

Video parameters behave like audio parameters:

- opacity  
- scale_x  
- scale_y  
- rotation  
- crop_x / crop_y / crop_w / crop_h  
- playback_rate  
- colour adjustments (future)  

Automation lanes work identically to audio.

Gesture/value-stream control (via hardware) also works normally.

---

# 8. Diagnostics for Video

From `23-diagnostics-and-performance.md`, video hooks include:

- decode time metrics  
- dropped frames  
- cache misses  
- proxy generation progress  
- GPU upload timing  
- frame-analysis node timing  

Aura may highlight problematic clips or nodes.

---

# 9. ARA-Style And Future Video Analytics

Although not needed for v1, the architecture supports:

- per-frame analysis plug-ins  
- face/object detection  
- motion analysis  
- beat alignment based on scene cuts  
- “smart sync” tools  
- multi-track video alignment (like multi-camera mode)

ARA-video plug-ins act as:

```
VideoAnalysisNode → produces metadata → stored in Pulse
```

Not required for v1, but fully supported by the model.

---

# 10. UX Summary

### Arrangement
- video lanes appear below audio lanes  
- thumbnails generated on demand  
- hover scrubbing  
- ghost render indicators for heavy edits  

### Video Preview Window
- movable, dockable  
- persists per screen set  
- supports safe fallback resolution on slow GPUs  

### Clip Inspector
- video metadata  
- transforms  
- colour adjustments  
- rate/stretching controls  

### Browser
- video assets shown with frame thumbnails  
- duration/timecode indicators  

---

# 11. File Formats & Containers

Supported for v1 (read-only, render output):

- QuickTime (MOV, MP4)  
- MP4/H.264  
- ProRes (if platform allows)  
- MKV (internal use only)  

Proxy formats:
- H.264 low-res  
- raw frame buffers for GPU  

Export:
- WAV or AIFF + MP4/MOV  
- optional separate stems  

---

# 12. IPC Responsibilities

## 12.1 Pulse → Signal
- request frame at timestamp  
- update preview region  
- request proxy generation  
- request decode pipeline changes  

## 12.2 Signal → Pulse
- decoded frame ready  
- proxy progress  
- decode error  
- cache exhaustion events  

## 12.3 Pulse ↔ Aura
- preview frame handles  
- clip thumbnails  
- status updates  
- inspector parameter updates  

---

# 13. Summary

This architecture ensures Loophole has a **modern, extensible, reliable video engine**, including:

- sample-accurate sync,  
- robust proxy system,  
- multilayer video lanes,  
- powerful clip and transform pipelines,  
- offline rendering pipeline with unified model,  
- diagnostics and performance tracking,  
- future-proofing for ARA-style video analysis,  
- and seamless integration with existing track/lane/node/clip systems.

Loophole starts as a *world-class soundtrack/video-scoring environment*  
and has the foundations to expand into a more complete video solution over time.
