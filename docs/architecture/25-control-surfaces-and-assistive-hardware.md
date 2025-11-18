# Control Surfaces & Assistive Hardware Architecture

This document defines the architecture for **control surfaces**, **MIDI controllers**, **DAW controllers**, and all **assistive hardware** connected to Loophole.  
It introduces a universal capability model, dynamic mapping engine, Composer integration, Pulse-driven control logic, and deep context awareness.

It also defines the responsibilities between:
- **Signal** (device detection & low-latency I/O)  
- **Pulse** (mapping logic & state machine)  
- **Aura** (UI presentation & feedback)  
- **Composer** (global knowledge & device intelligence)

This architecture builds on:
- `03-signal.md`
- `04-aura.md`
- `05-composer.md`
- `10-parameters.md`
- `11-node-graph.md`
- `16-timebase-tempo-and-groove.md`
- `19-automation-and-modulation.md`
- `20-midi-architecture.md`

---

## 1. Goals

Control surfaces must:
- **just work immediately** (zero setup)
- adapt their behaviour to context (Arranger, Mixer, Launcher, Plugin UI)
- use Composer-driven knowledge to provide intelligent default mappings
- support deep customisation without requiring scripting
- handle a huge variety of devices (MIDI, HID, OSC, MCU/HUI, custom protocols)
- be resilient to hot-plug/unplug events
- allow learning/overrides that persist per machine & per project
- allow full hardware feedback (LEDs, displays, RGB pads)
- support multiple devices simultaneously
- unify note-playing devices and control devices into one consistent framework

---

## 2. Hardware Device Model

Pulse stores all hardware as:

```
HardwareDevice {
  deviceId;
  vendor;
  product;
  transport;       // midi, hid, osc, combined
  capabilities: DeviceCapabilities;
  ports[];
  profiles[];      // mappings from Composer
  userOverrides[];
  activeProfile;
  status;
}
```

### 2.1 DeviceCapabilities

```
DeviceCapabilities {
  keys[];
  pads[];
  faders[];
  encoders[];
  buttons[];
  grids[];
  leds[];
  displays[];
  touchStrips[];
  ribbons[];
  wheels: { pitchBend?, modWheel? };
  mpeZones?;
  transportGroup?;
  motorisedFaders?;
  specialModes[];
  sysexSupport;
  hidDescriptor?;
}
```

Pulse treats all devices uniformly regardless of backend protocol.

---

## 3. Device Discovery (Signal)

Signal is responsible for:
- enumerating hardware on startup
- detecting plug/unplug events
- classifying endpoints (MIDI, HUI, MCU, HID, OSC)
- providing low-level input streams with timestamps
- low-latency forwarding of raw control messages to Pulse

Signal never performs mapping or semantic interpretation.

Signal may handle:
- MIDI parsing  
- OSC decoding  
- HID report decoding  
- Sysex identity queries  

Signal sends Pulse a **HardwareDeviceDescriptor** when a device appears.

---

## 4. Device Fingerprinting (Pulse + Composer)

Once Pulse receives a new device descriptor:
1. Pulse constructs a **fingerprint packet** describing every capability the device appears to expose.
2. Pulse sends fingerprint to Composer.
3. Composer responds with:
   - known device profile?  
   - device class (Keyboard, Pad Controller, MCU, Grid Surface, etc)  
   - recommended mappings  
   - recommended context modes (Arranger/Mixer/Plugin/Launcher)  
   - known behaviour quirks  
   - LED/display protocol knowledge  
   - sysex DAW mode handshake strings (if applicable)

This gives Loophole instant plug-and-play correctness.

Composer collects anonymised statistics about how users customise each device, improving future mappings.

---

## 5. Mapping System (Pulse)

Pulse owns all mapping logic:

```
MappingRule {
  ruleId;
  deviceId;
  sourceControl;      // pad, encoder, fader, CC, HID report element
  target;             // ParameterId | ActionId | MixerChannel | ClipAction | PluginParam
  mode: absolute | relative | toggle | step | momentary | delta;
  scaling?;
  deadzone?;
  feedback?;          // LED/colour/text/display behaviour
  context: Arranger | Mixer | Launcher | Plugin | ClipEditor | Global;
}
```

Mappings may be:

- **default** (provided by Composer)
- **contextual** (different depending on view)
- **user overrides**
- **project-local overrides** (stored in project metadata)
- **dynamic** (e.g. plugin ParameterPage switching)

Pulse evaluates mapping rules to produce:

- **Control Intents** → forwarded to the engine or UI  
- **Feedback Intents** → forwarded to Signal for LEDs/Displays

Pulse handles all mode switching and conflict resolution.

---

## 6. User Override & Learning System

Loophole’s learning system must be “super lightweight”:

**Learn Mode:**
1. User clicks “Learn”
2. Moves any hardware control
3. Pulse detects the active parameter/target in UI context
4. Pulse creates a MappingRule override
5. Composer is informed (to improve global knowledge)

Overrides stored as:

```
UserOverrideMapping {
  deviceId;
  overrides[];
  createdAt;
}
```

Overrides persist per-machine (tracked by hardware-ID groups) and per-project when relevant.

No scripting is required for common use-cases.

---

## 7. Context Awareness

Pulse automatically switches mapping sets based on context signals emitted by Aura or Pulse itself.

Contexts include:

- Arranger View  
- Mixer View  
- Plugin UI View  
- Clip Editor (Piano Roll / Audio Editor)  
- Launcher View  
- Focused Window (Plugin)  
- Focused Track  
- Global Mode  

Each context has:
- its own profile  
- its own mapping rules  
- its own visual feedback behaviour  
- its own “default bank” of parameters

Exported as:

```
ActiveContext {
  id;
  activeTrack?;
  activeClip?;
  activePlugin?;
  mixerBank?;
  launcherScene?;
}
```

Context switching is instant and does not interrupt hardware feedback.

---

## 8. Hardware Feedback (Pulse → Signal)

Pulse generates **Feedback Intents**:

```
FeedbackIntent {
  deviceId;
  targetControl;
  colour?;
  value?;
  text?;
  meterLevel?;
}
```

Signal translates these into:
- MIDI CC messages  
- RGB pad commands  
- sysex display updates  
- MCU/HUI display messages  
- HID output reports  

Signal’s job is purely delivery; Pulse decides all behaviour.

---

## 9. Plugin Parameter Interaction

When a plugin UI is focused:
- encoders → sorted list of visible parameters  
- bank switching updates parameter sets  
- faders → high-priority parameters  
- pads may trigger plugin-specific actions (if profile supports it)

Pulse makes plugin parameter lists available using:

```
PluginParamPage {
  pageId;
  parameters[];  // ordered list of ParameterIds
}
```

Composer provides metadata for well-known plugins:
- common groupings  
- sensible page names (“Oscillator”, “Filter”, “Envelopes”)  
- preferred hardware parameter arrangement  

---

## 10. Grid & Pad Controllers

Grid devices (Launchpad, Push, APC40, Maschine) can operate in:

- **Note Mode**
- **Drum Grid Mode**
- **Clip Launcher Mode**
- **Sequencer Mode**
- **Mixer Mode**
- **Plugin Mode**

Pulse switches modes based on active context.

LED feedback:
- track colours  
- clip states  
- scenes  
- mute/solo  
- step sequencer steps  
- plugin parameter levels  

Composer provides known pad-grid layouts for popular devices.

---

## 11. Transport Control

Transport controls (play/stop/record/loop) receive special low-latency routing:

- Signal handles button → transport intent
- Pulse confirms & triggers transport action
- Pulse sends LED feedback (e.g. “Record Armed”)
- Signal ensures input latency is minimal

Transport control is always enabled globally.  
Context never disables it.

---

## 12. Multi-Device Support

Pulse supports multiple devices simultaneously:

- separate profiles per device
- routing arbitration
- merged displays (if needed)
- device-specific context awareness
- independent overrides

Examples:
- Keyboard for notes  
- Pad controller for launcher  
- Mixer controller for faders  
- Secondary device controlling plugin  

Pulse manages the combined mapping set.

---

## 13. Matchmaking with Tracks, Mixer, Plugins

Pulse dynamically binds devices to active targets:

### Piano / MIDI Keyboard
- auto-binds to selected Track’s first instrument  
- MIDI routed automatically  
- MPE supported if device supports it  

### Mixer Surfaces
- maps faders to visible bank of tracks  
- bank switching follows track scroll  
- motorised faders receive position updates  

### Pad Controllers
- grid = Launcher clips  
- pads = drum pads if track contains DrumLane or sample racks  
- colourisation from clip/track colour  

### Plugin Controllers
- encoders = plugin parameters  
- displays show parameter values  

---

## 14. Device Profiles

Device profiles consist of:

```
DeviceProfile {
  profileId;
  deviceMatch;
  defaultMappings[];
  contextMappings[];
  startupMessages[];
  heartbeat?;
  sysexProtocol?;
  ledProtocol?;
  displayProtocol?;
}
```

Profiles are sourced from Composer and cached locally in Pulse.

Pulse may combine:
- vendor profile  
- community-enhanced profile  
- user overrides  

---

## 15. Hotplug / Disconnect Behaviour

When device connects:
- Pulse builds device  
- fingerprints via Composer  
- loads profile  
- applies default context  
- enables mappings  

When device disconnects:
- Pulse marks device unavailable  
- preserves mappings  
- removes feedback  
- avoids dangling actions  

If reconnected:
- restored seamlessly  
- per-machine aliases ensure naming stability  

---

## 16. Signal Responsibilities

Signal handles:
- raw MIDI  
- raw OSC  
- raw HID  
- sysex identity  
- transport triggers  
- low-latency MPE  
- output dispatch for LEDs/displays  

Signal must forward events to Pulse within a few ms.

Pulse must handle high-frequency CC decimation for Gestures (see `ipc` docs).

---

## 17. IPC Overview (Pulse ↔ Signal ↔ Aura)

### Signal → Pulse
- device connected  
- raw control message  
- HID packet  
- sysex identity  
- MIDI clock  

### Pulse → Signal
- feedback intents (LEDs, displays)  
- sysex control messages  
- mode-change messages  
- MPE configuration  

### Pulse ↔ Aura
- context change notifications  
- focused track/clip/plugin  
- UI state updates for learn mode  
- list of mappings & overrides  

---

## 18. Versioning & Persistence

Pulse saves:
- device registry  
- user overrides  
- Composer profile cache  
- per-machine hardware maps  
- per-project mappings (optional)

Versions allow:
- context-specific mapping sets  
- alternate hardware setups  
- project templates with device layouts  

---

## 19. Future Systems

The system is designed to support:

- fully scriptable control surfaces (Lua or plugin API)
- multi-device choreography (coordinated pad lighting)
- high-resolution HID scan rates  
- audio-rate modulation from hardware  
- OSC controllers for immersive setups  
- VR/AR control surfaces  
- AI-assisted mapping generation via Composer  

---

## 20. Summary

Loophole’s control surface architecture provides:

- plug-and-play with zero configuration  
- Composer-powered device intelligence  
- unified capability model for all hardware  
- context-aware mapping logic  
- learn mode & override system  
- deep plugin-integration capabilities  
- full LED/display feedback  
- multi-device support  
- deterministic Signal-level I/O  
- robust hotplug handling  
- future extensibility for advanced surfaces  

This ensures Loophole feels “instrumental” rather than “configured”:  
hardware becomes a natural extension of the DAW, not a barrier to using it.
