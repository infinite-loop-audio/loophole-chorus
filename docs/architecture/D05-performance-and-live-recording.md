# Performance & Live Recording Architecture

This document defines the architecture for **live performance, clip launching,
scene sequencing, and recording workflows** in Loophole. It describes how the
launcher, arrangement, transport, and engine interact during realtime
performance and recording operations.

It covers:

- clip and scene triggering,
- follow actions and scene sequencing,
- live quantisation,
- live-safe and cohort-safe DSP behaviour,
- loop / overdub / punch flows,
- retrospective MIDI and audio recording,
- multitrack recording,
- performance playthrough preview,
- launcher/arrangement unification.

It complements:

- `06-processing-cohorts-and-anticipative-rendering.md`
- `14-channel-routing-and-send-return.md`
- `22-editor-architecture.md`
- `19-comping-architecture.md`
- `12-mixer-and-channel-architecture.md`
- Transport IPC domains (Pulse/Signal)
- `13-clip-launcher.md`

---

# 1. Goals

Loophole must provide a **first-class live performance environment** with:

1. **Predictable quantised performance**
   - Sample-accurate clip and scene switching.
   - Clear countdown to trigger.

2. **Safe realtime behaviour**
   - No stalls or audible glitches when switching clips/scenes.
   - Pre-buffering and anticipative warm-up.

3. **Robust and flexible recording workflows**
   - Multitrack record.
   - Loop and overdub modes.
   - Punch in/out.
   - Retrospective MIDI and audio capture.

4. **Scene-based composition**
   - Scenes act as clip collections.
   - Follow actions for automatic sequencing.
   - Scene playlists and playthrough previews.

5. **Fluid movement between launcher and arrangement**
   - Same transport and timing.
   - Drag clips between views.
   - Commit scenes to arrangement.

6. **Consistency and clarity**
   - The user always knows what will happen when they trigger something.

---

# 2. Core Concepts

## 2.1 Dual Timeline Model

Loophole operates with two synchronised but independent views:

### Arrangement Timeline
- Traditional linear sequence.
- Contains the final composition.

### Launcher Timeline
- Grid of clips organised into **Scenes**.
- Non-linear triggering, loop-based.

Both share:
- one global transport,
- one timebase,
- one tempo system.

## 2.2 Performance Quantisation

Every trigger (clip, scene, punch, follow action) may be quantised to:
- bar,
- beat,
- half-beat,
- grid division,
- scene boundary,
- or none (immediate).

Quantisation may be:
- global,
- per-scene,
- per-clip.

## 2.3 Performance Cohorts

From `06-processing-cohorts-and-anticipative-rendering.md`:

- Channels involved in live performance must remain in the **realtime cohort**.
- Scene/clip switches must not cause blocking reallocation.
- Anticipative scheduling must pre-warm the required buffers and states.

Pulse determines safe boundaries and schedules warm-up windows.

---

# 3. Clip Launching

## 3.1 Clip Trigger Specification

```
ClipTrigger {
  clipId;
  quantise;            // global | bar | beat | division | none
  behaviour: start | stop | retrigger | legato;
  launchVelocity?;
}
```

Pulse schedules the trigger and informs Signal of the exact sample to begin at.

## 3.2 Legato Launch Mode

When switching clips on a lane:
- new clip resumes from the musical position of the old clip,
- timeline continuity is preserved,
- used for expressive, instrument-like switching.

## 3.3 Clip Stop Behaviour

Clip stopping may be:
- immediate,
- quantised,
- fade-out,
- section-end.

## 3.4 Crossfades Between Clips

Signal applies configurable crossfades where transitions would create discontinuities.

---

# 4. Scene System

Scenes define **rows of clips** that launch together.

### Scene Trigger Model

```
SceneTrigger {
  sceneId;
  quantise;
  mode: full | partial | playlist | followAction;
}
```

Behaviour:

- Starting a scene stops active clips (unless overridden).
- All clips in the scene begin together.
- Tracks without clips continue previous clip or silence.

---

# 5. Follow Actions

Follow actions provide **automatic navigation** between scenes or clips.

### Follow Action Specification

```
FollowAction {
  next: sceneId | clipId | random | repeat | stop;
  afterLoops?;
  probability?;
  constraints?: { noRepeat?, exclude?[], only?[] };
}
```

Follow action behaviour must be deterministic and stable under:
- scene playlist playback,
- launcher/arrangement switching,
- offline renders.

---

# 6. Scene Playthrough Preview

As discussed in design sessions:

- Loophole can play **each scene sequentially**,
- each scene lasts the duration of its longest clip,
- shorter clips loop inside their scene,
- the sequence is played as a temporary timeline,
- arrangement is untouched unless committed.

Useful for auditioning entire tracks without arranging.

---

# 7. Record Modes

Loophole supports:

## 7.1 Linear Recording
- Multiple armed tracks.
- Perfect alignment with latency compensation.

## 7.2 Loop Recording
Two variants:

- **Takes Mode**: each loop → new lane for comping.
- **Layers Mode**: each loop → new overlay layer.

## 7.3 MIDI Overdub
- Merge-mode for existing clips.
- Quantise-on-input optional.
- Captures CC/expression if enabled.

## 7.4 Audio Overdub (Layered)
- Creates new audio layers within Clip Pipelines.
- Phase-safe merging possible later via pipeline steps.

## 7.5 Punch In/Out
Punch points scheduled with sample accuracy:

- punch in only,
- punch out only,
- punch region,
- pre/post roll settings.

---

# 8. Retrospective Recording

## 8.1 MIDI Retrospective

Signal maintains always-on buffers per MIDI source:

```
MidiRetroBuffer {
  events[];
  maxDuration;
}
```

Pulse interprets these buffers to create:
- new clips,
- merged clips,
- quantised or raw captures.

## 8.2 Audio Retrospective (Configurable)

When enabled:
- Signal records input channels to circular disk/RAM buffers,
- Pulse can extract new takes,
- used for “I played something great but wasn’t recording”.

---

# 9. Live Quantisation & Timing

## 9.1 Input Timing Correction

Pulse applies:
- input latency compensation,
- optional quantise-on-input,
- optional groove matching.

## 9.2 Monitoring Path

Monitoring must:
- run exclusively in the realtime cohort,
- never be affected by anticipative tasks,
- bypass unnecessary processing unless explicitly configured.

## 9.3 Smart Quantisation (Future)

Composer-assisted quantisation:
- learns performer groove,
- adjusts timing adaptively,
- ensures expressive quantisation.

---

# 10. Cohort Interaction & Safety

Performance requires:

1. **No DSP thread manipulations during a trigger event.**
2. **Clip switching must be pre-warmed**, including:
   - plugin state initialisation,
   - clip window buffer fills,
   - sidechain routing validation.

3. **Launcher state changes must never block.**

Pulse orchestrates:
- preloading,
- cohort assignment,
- crossfade planning,
- safe trigger timing.

Signal enforces:
- no blocking disk I/O,
- realtime-safe buffer transitions,
- sample-accurate start times.

---

# 11. Multitrack Recording

## 11.1 Audio Recording

Signal allocates:
- track-specific write streams,
- background disk I/O workers,
- preallocated buffers.

Pulse:
- generates takes,
- updates comp lanes,
- provides clip metadata.

## 11.2 MIDI Recording

MIDI is recorded:
- as raw timestamped events,
- with post-process quantisation options,
- with CC/expression layer capture.

---

# 12. Launcher / Arrangement Integration

## 12.1 Unified Transport

Arrangement and Launcher share:
- tempo,
- playhead,
- transport state,
- timebase.

## 12.2 Moving Clips Between Views

Dragging clip:
- preserves ID,
- moves or duplicates accordingly,
- updates lane routing,
- performs content validation.

## 12.3 Committing Scenes to Arrangement

Pulse converts scenes to arrangement clips:
- aligns according to scene length,
- handles loop assumptions,
- preserves lane structure.

---

# 13. IPC Responsibilities

### Aura → Pulse
- `clip.trigger`
- `scene.trigger`
- `scene.followAction.update`
- `launcher.quantise.update`
- `recordMode.set`
- `record.arm`
- `record.disarm`
- `retrospective.commit`
- `scene.playthrough.request`

### Pulse → Aura
- `trigger.scheduled`
- `record.takeCreated`
- `record.bufferStatus`
- `launcher.state`
- `scene.playthrough.preview`

### Pulse → Signal
- `performance.scheduleTrigger`
- `performance.preloadClip`
- `record.startTrackStream`
- `record.endTrackStream`
- `retrospective.flush`

### Signal → Pulse
- `trigger.executed`
- `record.underrun`
- `retrospective.ready`

---

# 14. UX Expectations

In performance mode, Aura provides:

- clip trigger countdown overlays,
- active scene highlighting,
- follow-action visualisation,
- overdub indicators,
- quantisation grid indicators,
- multitrack recording arm/monitor status,
- low-latency monitoring paths,
- hardware controller integration (pads, faders, scenes),
- clear state changes without modal interruptions.

---

# 15. Future Extensions

- Scene Playlist scripting  
- Live macros (gesture-driven)  
- Conductor/Tempo Cue view  
- Multi-user networked performance  
- Scene morphing  
- DJ-style crossfade decks  
- Machine-learning clip suggestions  
- Touch/VR performance modes  

---

# 16. Summary

Loophole’s performance architecture provides:

- sample-accurate clip and scene triggering,
- safe and predictable quantised performances,
- robust multitrack, loop and overdub recording,
- retrospective capture,
- seamless launcher/arrangement integration,
- deep cohort-aware engine behaviour,
- expressive live workflows.

This enables Loophole to function simultaneously as:
- a world-class linear DAW, and
- a powerful realtime performance instrument.
