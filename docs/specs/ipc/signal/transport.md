# Signal IPC – Transport Domain

This document defines the **Transport** IPC domain between Pulse and Signal.

The Transport domain is responsible for:

- low-level control of playback state in the Signal engine,
- sample-accurate positioning and seeking,
- loop and pre-roll settings at the engine level,
- reporting current playback position and state back to Pulse.

The Pulse `transport` domain operates at the **musical / project** level  
(bars/beats, sections, scenes).  
The Signal `transport` domain operates at the **engine / sample** level.

Aura never talks directly to Signal; all commands flow **Pulse → Signal**, and
all engine events flow **Signal → Pulse**.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Transport domain:

- starts and stops engine playback,
- seeks the engine’s playback head to a specific sample position,
- sets loop regions for the engine’s internal transport,
- controls record state (arm/record/punch) at the engine level,
- reports playback state and current position to Pulse at a low but regular rate.

### 1.2 Non-Goals

The Transport domain does **not**:

- define the musical timebase (tempo, time signature, tempo map),
- decide where to seek in musical terms (bars/beats/sections/scenes),
- manage project sections, scenes or clip-launcher behaviour,
- decide which Tracks/Channels/Lanes should actually produce audio.

Those concerns belong to Pulse (`timebase`, `transport`, `launcher`, etc.) and
the graph configuration (`signal.graph`).

Signal’s transport is a **dumb but precise engine** that obeys Pulse’s
instructions.

---

## 2. Coordinate Systems & Timebase

### 2.1 Sample Position

All Signal transport positions are expressed in **project sample frames**:

- `positionSamples` is a 64-bit integer (`int64`) representing the sample index
  from project start (0 = project start).
- Sample rate is defined by the engine configuration (`signal.engine` +
  `signal.hardware` effective sample rate).

Pulse is responsible for:

- converting musical positions (bars/beats/timecode) into `positionSamples`,
- maintaining the tempo map and ensuring consistency between musical time and
  sample positions.

### 2.2 Playback Position Updates

Signal emits periodic **position updates**:

- Pulse uses these to:
  - keep Aura’s timeline cursor in sync,
  - drive playhead animations,
  - check alignment between engine state and model state.

Update rate is implementation-defined, but typical values are 20–60 Hz. The rate
should be configurable or adaptive to avoid overload.

---

## 3. Message Flow Overview

All messages in this domain are:

- Commands: **Pulse → Signal**
- Events: **Signal → Pulse**

A typical lifecycle:

1. Pulse configures the engine (`signal.engine`, `signal.hardware`, `signal.graph`).
2. Pulse sets loop region and pre-roll, if required.
3. Pulse calls `transport.seek` to set initial position.
4. Pulse calls `transport.play` to start playback.
5. Signal emits `transport.stateChanged` and periodic `transport.positionUpdate`.
6. Pulse may call `transport.stop`, `transport.pause`, or further `transport.seek`
   as needed.

---

## 4. Commands (Pulse → Signal)

### 4.1 `transport.play`

Start playback from the current engine playback position.

**Request**

```json
{
  "command": "transport.play",
  "options": {
    "startFromSeekTarget": false
  }
}
```

Semantics:

- If `startFromSeekTarget` is `true` and there is a pending seek position in an
  implementation-specific buffer, the engine may start from that target; this is
  optional and may be ignored.
- If the engine is already playing, this command is idempotent and should not
  alter the current position.

---

### 4.2 `transport.pause`

Pause playback at the current position.

**Request**

```json
{
  "command": "transport.pause"
}
```

Semantics:

- Playback stops, but the current position is retained.
- A subsequent `transport.play` resumes from that position.
- If the engine is already paused or stopped, this command is idempotent and
  should not alter the current position.

---

### 4.3 `transport.stop`

Stop playback and move the engine into a “stopped” state.

**Request**

```json
{
  "command": "transport.stop"
}
```

Semantics:

- The exact distinction between “paused” and “stopped” is implementation
  defined, but typical behaviour:
  - `pause` – playback halted, engine ready to resume without re-arming,
  - `stop` – playback halted, engine may reset internal pipelines, flush
    buffers, and require explicit `play` to resume.
- The position is normally retained; Pulse can explicitly seek elsewhere using
  `transport.seek`.

---

### 4.4 `transport.seek`

Move the engine playback position to a given sample frame.

**Request**

```json
{
  "command": "transport.seek",
  "positionSamples": 123456789
}
```

Semantics:

- Position is specified in project sample frames, relative to project start.
- If called while playing, behaviour is:
  - seek to position and continue playing from the new position as soon as
    possible, or at a buffer boundary,
  - Signal should keep jitter as low as possible and emit updated
    `transport.stateChanged` and `transport.positionUpdate` accordingly.
- If called while stopped/paused, only the position is updated.

Pulse is responsible for aligning seek positions to musical intentions (e.g.
bars, beats, sections).

---

### 4.5 `transport.setLoopRegion`

Define or clear a loop region in engine sample coordinates.

**Request**

```json
{
  "command": "transport.setLoopRegion",
  "enabled": true,
  "startSamples": 96000,
  "endSamples": 192000
}
```

Semantics:

- When `enabled` is `true`, engine playback should loop between
  `startSamples` (inclusive) and `endSamples` (exclusive).
- If `enabled` is `false`, engine should run linearly until stopped.
- Pulse must ensure that `endSamples > startSamples` when enabling.
- Signal may clamp or adjust the region if it detects invalid arguments, and
  must inform Pulse via error response or a warning event.

Loop behaviour for anticipative rendering is handled in `signal.graph` and
internal engine logic; the Transport domain simply defines the region.

---

### 4.6 `transport.setPreRoll`

Define a pre-roll duration for record or playback.

**Request**

```json
{
  "command": "transport.setPreRoll",
  "enabled": true,
  "durationSamples": 48000
}
```

Semantics:

- If enabled, the engine should begin playback `durationSamples` before the
  requested start position when entering play/record, while Pulse may still
  treat the main logical start position differently.
- This is primarily for record workflows; the detailed interaction with record
  arming is defined by Pulse and recording domains.

---

### 4.7 `transport.setRecordState`

Control engine-level record state (arm and active recording).

**Request**

```json
{
  "command": "transport.setRecordState",
  "armed": true,
  "recording": false
}
```

Semantics:

- `armed = true` indicates that the engine should be ready to record from
  armed inputs once playback begins.
- `recording = true` indicates that recording should be actively capturing
  data during playback.
- Pulse coordinates this with track and lane-level record arming and routes;
  Signal merely executes the record state and routes audio/MIDI to record
  buffers.

Note: Detailed record routing, track arming per channel, and file sinks are
covered in other specs; here we only define the engine’s global record state.

---

## 5. Events (Signal → Pulse)

### 5.1 `transport.stateChanged`

Signals a change in engine transport state.

**Payload**

```json
{
  "event": "transport.stateChanged",
  "state": "playing", // "stopped" | "paused" | "playing"
  "positionSamples": 123456789,
  "loop": {
    "enabled": true,
    "startSamples": 96000,
    "endSamples": 192000
  },
  "record": {
    "armed": true,
    "recording": false
  }
}
```

Notes:

- Emitted whenever the engine transitions between playing/paused/stopped.
- Includes the current effective position and loop/record states for Pulse to
  reconcile with its model.

---

### 5.2 `transport.positionUpdate`

Periodic low-frequency updates of the current engine position.

**Payload**

```json
{
  "event": "transport.positionUpdate",
  "state": "playing", // "stopped" | "paused" | "playing"
  "positionSamples": 123456789
}
```

Semantics:

- Emitted at a regular interval while the engine is playing.
- Pulse uses these to:
  - update Aura’s playhead,
  - detect desync if any,
  - drive time-dependent UI features that do not require sample-accurate timing.

The update rate should be chosen to balance responsiveness and IPC overhead.

---

### 5.3 `transport.loopRegionChanged`

Loop region has changed, whether due to Pulse commands or internal adjustments
(e.g. clamping).

**Payload**

```json
{
  "event": "transport.loopRegionChanged",
  "enabled": true,
  "startSamples": 96000,
  "endSamples": 192000
}
```

Notes:

- May be emitted in response to `transport.setLoopRegion`, or if the engine
  must adjust the region (e.g. due to project length constraints).

---

### 5.4 `transport.recordStateChanged`

Record state has changed.

**Payload**

```json
{
  "event": "transport.recordStateChanged",
  "armed": true,
  "recording": true
}
```

Notes:

- Emitted whenever the engine starts or stops active recording, or when the
  global armed state changes.

---

## 6. Error Handling

Transport commands follow the general IPC error envelope described in the
Envelope spec. Typical error cases include:

- seeking to a negative or out-of-range position,
- invalid loop region (end ≤ start),
- attempting to record without configured hardware devices,
- attempting `play` while engine is not running (not configured / no device).

Example error response:

```json
{
  "error": {
    "code": "invalidLoopRegion",
    "message": "Loop endSamples must be greater than startSamples.",
    "domain": "transport",
    "details": {
      "startSamples": 200000,
      "endSamples": 150000
    }
  }
}
```

Signal should prefer:

- clamping or correcting values when safe to do so, and
- reporting corrections via events (`transport.loopRegionChanged`), rather than
  failing hard.

---

## 7. Sync & Consistency with Pulse

Pulse is the **source of truth** for musical time and transport intent.
Signal is the **source of truth** for actual playback position in samples.

Key rules:

- Pulse must not assume the engine is playing or stopped solely based on
  internal model; it must treat `transport.stateChanged` as authoritative.
- Pulse should align its internal concept of current musical position with
  `positionSamples` from Signal, using the tempo map from the Timebase domain.
- Any higher-level constructs (Sections, Scenes, playthrough modes) are
  implemented in Pulse and then translated into a series of:
  - `transport.seek`,
  - `transport.play`,
  - `transport.stop`,
  - `transport.setLoopRegion` calls.

If Pulse detects drift between expected and actual positions (e.g. due to
engine restarts or errors), it may:

- issue a corrective `transport.seek` and `transport.play`,
- or re-synchronise by rebuilding the graph and re-issuing commands.

---

## 8. Relationship to Other Signal Domains

- **signal.engine**  
  Transport commands are only valid when the engine is configured and running.
  If the engine is stopped or in an error state, transport commands may
  respond with errors or be ignored.

- **signal.graph**  
  The graph defines which Channels and Nodes respond to the transport state;
  transport simply moves the playhead and controls playback.

- **signal.media**  
  Media decoding and streaming must follow the transport position; the Media
  domain may pre-decode or cache content based on recent and upcoming
  positions.

- **signal.diagnostics**  
  Performance diagnostics (underruns, callback jitter) may be correlated with
  transport state and position, but are reported separately.

---

## 9. Extensibility

Possible future extensions to this domain (reserved for backlog):

- Scrub modes (slow, sound-on-scrub, shuttle).  
- Timecode sync (e.g. SMPTE-based position reporting).  
- Multiple transport contexts (e.g. audition vs main timeline).  
- Per-Track or per-Cohort transport views for advanced rendering workflows.

Such features should be designed to fit within the existing model: a simple,
sample-based engine playback controller under Pulse’s direction.
