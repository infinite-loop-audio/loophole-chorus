# Signal IPC – Gesture Domain

This document defines the **Gesture** IPC domain between Pulse and Signal.

The Gesture domain provides a **low-latency, high-rate control path** for
time-varying parameter changes arising from user input (e.g. knob drags, fader
moves, XY pads), without flooding the normal JSON command channel.

Conceptually:

- **Pulse** coordinates gesture *sessions* (who controls what, when, and how).  
- **Signal** provides a **gesture stream endpoint**:
  - high-rate value packets from Aura,
  - mapped to one or more engine parameters,
  - with optional low-rate tap back to Pulse.

Aura never talks to Signal directly for control plane (open/close sessions);
it always goes via Pulse.  
But **for data**, Aura may connect directly to Signal using a dedicated
gesture stream channel.

---

## 1. Responsibilities & Non-Goals

### 1.1 Responsibilities

The Signal Gesture domain:

- accepts gesture session definitions from Pulse:
  - which parameter(s) are affected,
  - normalisation/domain information,
  - optional smoothing/lag behaviour,
- exposes **gesture stream endpoints**:
  - identified by `gestureSessionId`,
  - bound to one or more engine parameters (Node parameters, fader, send).
- decodes high-rate value packets and applies them to engine parameters in a
  real-time-safe way.
- optionally emits low-rate snapshots back to Pulse to keep the model in sync.

### 1.2 Non-Goals

The Gesture domain does **not**:

- decide when gestures start/stop (Pulse decides based on Aura input),
- own parameter metadata (that’s the Parameter/Node/Plugin domains),
- replace normal parameter set/get commands:
  - it provides a *temporary fast path*,
  - final parameter values are committed via Pulse’s model.

---

## 2. Overview: Gesture Lifecycle

The typical lifecycle of a gesture looks like this:

1. **Aura → Pulse (UI layer)**  
   Aura detects a user interaction and asks Pulse to start a gesture:
   - which parameter(s),
   - expected input domain (e.g. normalised 0–1),
   - smoothing expectations.

2. **Pulse → Signal (control plane)**  
   Pulse calls `gesture.openSession` on the Signal IPC control channel:
   - describing the target parameter(s),
   - giving a `gestureSessionId` (chosen by Pulse),
   - requesting a **stream endpoint**.

3. **Signal → Pulse**  
   Signal validates, binds parameters, and replies with:
   - acknowledgement,
   - optional stream parameters (packet format, recommended rate).

4. **Aura ↔ Signal (data plane)**  
   Aura obtains the `gestureSessionId` and any connection details from Pulse,
   then:
   - opens a direct gesture stream connection to Signal,
   - sends value packets at high rate (for the duration of the gesture),
   - optionally receives acknowledgements or low-rate mirror samples.

5. **Gesture end**  
   - Aura stops sending data and notifies Pulse that the gesture ended.  
   - Pulse:
     - finalises the parameter value in its model,
     - sends a normal parameter update to Signal if needed,
     - calls `gesture.closeSession` to tear down the engine-side mapping.

The Gesture domain here focuses on **Pulse ↔ Signal** control messages, and the
definition of the **data stream contract**, not the concrete transport.

---

## 3. Parameter Binding Model

A **gesture session** can target one or more engine parameters.

Each target is described as:

```json
{
  "targetId": "param:node:track:42:plugin:1:cutoff",
  "nodeId": "node:track:42:plugin:1",
  "parameterId": "param:cutoff",
  "mode": "absolute", // "absolute" | "relative"
  "scale": {
    "inputMin": 0.0,
    "inputMax": 1.0,
    "outputMin": 20.0,
    "outputMax": 20000.0,
    "curve": "linear" // "linear" | "log" | "exp" | future extensions
  }
}
```

- `targetId` is a Pulse-stable identifier for the parameter binding.
- `mode`:
  - `absolute` – values in the stream represent full parameter position.
  - `relative` – values represent deltas (e.g. -1 .. +1), to be applied relative
    to the current value.

Signal uses this mapping to:

- transform incoming gesture values into engine parameter values,
- handle support for smoothing, clamping, and range enforcement.

---

## 4. Commands (Pulse → Signal)

These commands travel over the **normal JSON IPC control channel** between
Pulse and Signal.

### 4.1 `gesture.openSession`

Request a new gesture session and obtain a stream endpoint.

**Request**

```json
{
  "command": "gesture.openSession",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "targets": [
    {
      "targetId": "param:node:track:42:plugin:1:cutoff",
      "nodeId": "node:track:42:plugin:1",
      "parameterId": "param:cutoff",
      "mode": "absolute",
      "scale": {
        "inputMin": 0.0,
        "inputMax": 1.0,
        "outputMin": 20.0,
        "outputMax": 20000.0,
        "curve": "log"
      }
    }
  ],
  "options": {
    "smoothing": {
      "enabled": true,
      "timeConstantMs": 10
    },
    "maxUpdateRateHz": 240,
    "mirrorToPulse": {
      "enabled": true,
      "rateHz": 30
    }
  }
}
```

**Response**

```json
{
  "replyTo": "gesture.openSession",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "stream": {
    "streamId": "gestureStream:signal:001",
    "codec": "simpleFloatV1",
    "maxUpdateRateHz": 240
  }
}
```

Semantics:

- `gestureSessionId` is chosen by Pulse and used consistently in Pulse/Aura.
- `streamId` and `codec` describe the **data plane** endpoint on Signal.
- `smoothing` hints allow Signal to apply lightweight parameter smoothing on
  the engine side, suitable for quick drags.
- `mirrorToPulse`:
  - if enabled, Signal will send decimated value snapshots back to Pulse via
    `gesture.mirrorUpdate` events.

Signal may reject the request (e.g. unknown node/parameter) with a standard IPC
error.

---

### 4.2 `gesture.updateTargets`

Update the target bindings for an existing session.

**Request**

```json
{
  "command": "gesture.updateTargets",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "targets": [
    {
      "targetId": "param:node:track:42:plugin:1:resonance",
      "nodeId": "node:track:42:plugin:1",
      "parameterId": "param:resonance",
      "mode": "absolute",
      "scale": {
        "inputMin": 0.0,
        "inputMax": 1.0,
        "outputMin": 0.1,
        "outputMax": 10.0,
        "curve": "exp"
      }
    }
  ]
}
```

Semantics:

- Allows complex UI gestures (e.g. modifier keys, macro moves) to remap what a
  given stream controls mid-gesture.
- Signal should apply updates as soon as reasonably safe; some audio callback
  jitter is acceptable.

---

### 4.3 `gesture.setOptions`

Update session options (smoothing, mirror rate) without changing targets.

**Request**

```json
{
  "command": "gesture.setOptions",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "options": {
    "smoothing": {
      "enabled": true,
      "timeConstantMs": 5
    },
    "mirrorToPulse": {
      "enabled": true,
      "rateHz": 60
    }
  }
}
```

---

### 4.4 `gesture.closeSession`

Terminate a gesture session and release any associated engine-side state.

**Request**

```json
{
  "command": "gesture.closeSession",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag"
}
```

Semantics:

- Signal should:
  - stop accepting new packets on the corresponding `streamId`,
  - finalise any internal smoothing or interpolation,
  - stop emitting mirror updates for that session.
- Pulse calls this once it has:
  - committed the final parameter value into its model,
  - optionally sent a canonical parameter update through the standard parameter
    domain.

---

## 5. Data Plane: Gesture Stream Protocol

The data plane is the **high-rate value stream** from Aura to Signal.

This document describes the **packet structure and expectations**, without
mandating a particular transport (e.g. separate IPC pipe, shared memory region,
WebSocket-like channel).

### 5.1 Stream Binding

Signal will expose a stream endpoint per `gestureSessionId` with:

- `streamId`: engine-side reference (may be shared across multiple sessions
  internally, but appears unique to Pulse/Aura),
- `codec`: describes packet format.

For the initial implementation, we define a **simple float codec**:

```json
"codec": "simpleFloatV1"
```

### 5.2 Packet Structure – `simpleFloatV1`

Each packet in the stream encodes:

- a **monotonically increasing sequence number**,  
- a **timestamp** (optional, implementation/hardware dependent),  
- one or more **value samples**.

Abstract JSON representation (for documentation purposes):

```json
{
  "seq": 123,
  "t": 1731938400.123,      // optional (seconds), or omitted for implicit timing
  "values": [
    {
      "targetIndex": 0,
      "value": 0.732
    }
  ]
}
```

Actual on-the-wire format is implementation-defined and may use:

- binary floats,
- compact framing,
- batched samples.

The important semantics:

- `targetIndex` refers to the Nth element in the current `targets` array for
  the session.
- `value` is a float in the `scale.inputMin`..`scale.inputMax` domain.
- Packets are **best-effort**:
  - Signal should tolerate dropped or out-of-order packets,
  - typically only the latest value(s) matter per callback.

### 5.3 Expected Rates & Behaviour

- UI-side (Aura) may send at **120–240 Hz** for smooth drags, but this is a
  guideline, not a requirement.
- Signal:
  - applies the latest available value during each processing block,
  - can downsample or interpolate as needed,
  - must not allocate or block in the audio callback while handling gesture
    updates.

If the stream stalls (packets stop), Signal simply holds the last known value
until the session is closed or a timeout is reached.

---

## 6. Events (Signal → Pulse)

These events go over the **normal JSON IPC control channel**.

### 6.1 `gesture.sessionOpened`

Session has been successfully opened and bound to a stream.

**Payload**

```json
{
  "event": "gesture.sessionOpened",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "stream": {
    "streamId": "gestureStream:signal:001",
    "codec": "simpleFloatV1",
    "maxUpdateRateHz": 240
  }
}
```

Pulse forwards the relevant stream information to Aura via its own IPC layer.

---

### 6.2 `gesture.sessionClosed`

Session has been closed, either by Pulse or internally by Signal.

**Payload**

```json
{
  "event": "gesture.sessionClosed",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "reason": "normal" // "normal" | "timeout" | "error"
}
```

Semantics:

- If `reason` ≠ `"normal"`, Pulse may need to reconcile:
  - last-known mirror values,
  - UI expectations (e.g. aborting a drag).

---

### 6.3 `gesture.mirrorUpdate`

Low-rate mirror of gesture values back to Pulse (if enabled).

**Payload**

```json
{
  "event": "gesture.mirrorUpdate",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "targets": [
    {
      "targetId": "param:node:track:42:plugin:1:cutoff",
      "value": 0.732
    }
  ]
}
```

Semantics:

- Emitted at a capped rate (e.g. 20–60 Hz), based on `mirrorToPulse.rateHz`.
- Pulse uses this to:
  - keep its model “nearly in sync” during long drags,
  - update UI elements driven from Pulse state (e.g. other linked controls),
  - reduce the gap between final commit and last engine state.

---

### 6.4 `gesture.warning`

Non-fatal issues related to a gesture session.

**Payload**

```json
{
  "event": "gesture.warning",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "code": "streamBackpressure",
  "message": "Gesture stream is receiving data faster than it can be consumed.",
  "details": {
    "droppedPackets": 12
  }
}
```

Examples:

- stream backpressure or packet drops,
- unknown `targetIndex` in data packets (e.g. out-of-date UI),
- smoothing clamping excessively to avoid zipper noise.

---

### 6.5 `gesture.error`

Serious error causing a session to be unusable.

**Payload**

```json
{
  "event": "gesture.error",
  "gestureSessionId": "gesture:pulse:track:42:cutoffDrag",
  "code": "invalidTarget",
  "message": "Gesture session target no longer exists.",
  "details": {
    "nodeId": "node:track:42:plugin:1"
  }
}
```

Semantics:

- Signal should:
  - stop applying incoming gesture values for this session,
  - eventually close the session with `gesture.sessionClosed` (`reason: "error"`).
- Pulse should:
  - stop routing gesture data from Aura to this session,
  - resolve the underlying cause (e.g. node removed, parameter changed).

---

## 7. Error Handling (Commands)

Gesture commands can fail synchronously for reasons including:

- unknown `gestureSessionId`,
- invalid target definitions (unknown `nodeId` or `parameterId`),
- unsupported `codec` or options.

Example error:

```json
{
  "error": {
    "code": "invalidTarget",
    "message": "Node or parameter not found for gesture target.",
    "domain": "gesture",
    "details": {
      "nodeId": "node:track:42:plugin:1",
      "parameterId": "param:cutoff"
    }
  }
}
```

Pulse must treat these as authoritative and:

- cancel the gesture session on its side,
- revert UI state if necessary,
- optionally fall back to a slower, non-streamed parameter edit.

---

## 8. Relationship to Other Signal Domains

- **signal.parameters / node / plugin**  
  Gesture sessions ultimately drive parameter values belonging to Nodes and
  plugin instances.

- **signal.transport & signal.graph**  
  Gesture-induced parameter changes are applied within the current graph and
  transport state; gesture streams must remain stable under graph mutations
  (or cleanly error/close).

- **signal.diagnostics**  
  Diagnostics may track:
  - gesture stream health,
  - smoothing performance,
  - any high-frequency control issues that impact CPU.

- **Pulse gesture domain**  
  The Pulse gesture domain is the *logical* gesture controller:
  - initiates sessions,
  - decides what parameters are bound,
  - receives mirror updates and finalises model changes.

Signal’s Gesture domain is the **engine-side executor** of those sessions.

---

## 9. Extensibility

Future extensions may include:

- multi-dimensional gesture codecs (XY pads, multi-finger touch surfaces),
- higher-order shape streams (e.g. curve parameters rather than point samples),
- coalesced gestures across multiple parameters with shared timing,
- explicit “record gesture as automation” endpoints on the Signal side.

These should remain compatible with the core design:

- control plane via JSON IPC (Pulse ↔ Signal),
- data plane via dedicated high-rate streams (Aura ↔ Signal),
- Pulse as the authority over which parameters are affected and how they are
  ultimately committed into the project model.
