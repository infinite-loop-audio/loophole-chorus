# Composer Architecture

This document defines the conceptual model and responsibilities of **Composer**,
Loophole’s shared knowledge metadata service. Composer aggregates publicly
shared, user-derived, and vendor-provided information about plugins, parameters,
semantic roles, categories and mappings in order to provide Loophole with:

- robust parameter remapping across plugin versions and formats,
- semantic mapping between different plugins,
- structured plugin classification and tagging,
- high-quality metadata for UI, automation and organisation.

Composer is an external web service; Pulse integrates with it through a local
knowledge layer. Composer provides “semi-truths”: suggestions, mappings, and
metadata with confidence values. Pulse remains the authoritative source of truth
for project state.

This is an architectural document. It describes concepts, responsibilities and
flows, not implementation details or API formats.

---

## Contents

- [1. Overview](#1-overview)
- [2. Goals](#2-goals)
- [3. High-Level Responsibilities](#3-high-level-responsibilities)
- [4. Core Data Model](#4-core-data-model)
  - [4.1 Plugin Identity](#41-plugin-identity)
  - [4.2 Parameter Identity](#42-parameter-identity)
  - [4.3 Semantic Roles](#43-semantic-roles)
  - [4.4 Plugin Categories and Tags](#44-plugin-categories-and-tags)
  - [4.5 User Actions as Signals](#45-user-actions-as-signals)
  - [4.6 Hardware Device Identity](#46-hardware-device-identity)
  - [4.7 Hardware Capability Fingerprints](#47-hardware-capability-fingerprints)
- [5. Client Integration (Pulse and Aura)](#5-client-integration-pulse-and-aura)
  - [5.1 Pulse Knowledge Layer](#51-pulse-knowledge-layer)
  - [5.2 Interaction with Parameter Aliasing](#52-interaction-with-parameter-aliasing)
  - [5.3 Interaction with Aura](#53-interaction-with-aura)
  - [5.4 Interaction with Hardware I/O](#54-interaction-with-hardware-io)
- [6. Use Cases](#6-use-cases)
  - [6.1 Plugin Version Updates](#61-plugin-version-updates)
  - [6.2 Switching Plugin Formats](#62-switching-plugin-formats)
  - [6.3 Plugin Replacement and Meaning-Based Mapping](#63-plugin-replacement-and-meaning-based-mapping)
  - [6.4 Plugin Organisation and Browsing](#64-plugin-organisation-and-browsing)
  - [6.5 Hardware Device Recognition and Role Mapping](#65-hardware-device-recognition-and-role-mapping)
  - [6.6 Cross-Machine Hardware Fallback](#66-cross-machine-hardware-fallback)
- [7. Sync, Caching and Offline Behaviour](#7-sync-caching-and-offline-behaviour)
- [8. Privacy and Consent](#8-privacy-and-consent)
- [9. Failure Modes and Fallbacks](#9-failure-modes-and-fallbacks)
- [10. Deterministic Behaviour Telemetry and Inference](#10-deterministic-behaviour-telemetry-and-inference)
  - [10.7 Hardware Profile Telemetry](#107-hardware-profile-telemetry)
- [11. Hardware Knowledge API](#11-hardware-knowledge-api)
- [12. Future Extensions](#12-future-extensions)

---

## 1. Overview

Composer is Loophole's shared knowledge metadata service. It maintains a
continuously improving model of:

- plugins (their identities, formats and versions),
- parameters (their names, types, ranges, roles and groupings),
- relationships between plugins (mapping across versions and formats),
- semantic equivalences across different plugins,
- plugin categories and user-driven classification,
- hardware devices (their identities, capabilities, and typical role mappings).

Composer is optional but enhances Loophole's resilience and ergonomics. When
available, it supports:

- smarter automation remapping,
- robust project loading after plugin updates,
- automatic tagging and organisation of plugin collections,
- semantic mapping when replacing plugins,
- better UX in browsing, categorisation and preset behaviour,
- intelligent hardware device recognition and role mapping suggestions,
- fallback recommendations when projects are opened on different hardware.

Composer suggestions are non-authoritative. Pulse combines them with local
aliasing, heuristics and user preferences to choose the final mapping.

---

## 2. Goals

Composer aims to:

- prevent loss of automation and parameter state when plugins change,
- reduce friction in plugin organisation and discovery,
- allow semantic operations across plugins (e.g. "map cutoff to cutoff"),
- provide a shared ecosystem of metadata continuously improved by user input,
- support long-term sustainability for projects spanning multiple plugin
  versions and formats,
- assist with hardware device recognition and role mapping across different
  machines and hardware configurations,
- provide intelligent fallback suggestions when hardware changes.

Composer must:

- be optional and non-blocking,
- have no access to project media or arrangement data,
- work entirely from plugin and parameter metadata,
- protect privacy and operate on anonymised signals,
- function gracefully offline with local caching,
- provide hardware recommendations without enforcing mappings,
- maintain global/community-level hardware knowledge separate from
  machine-specific configurations.

---

## 3. High-Level Responsibilities

Composer is responsible for maintaining and serving:

### **Plugin Metadata**
- canonical plugin family identifiers,
- per-format, per-version variant descriptors,
- vendor names, product names, internal IDs.

### **Parameter Metadata**
- names, short names,
- types, units, ranges,
- scaling behaviours,
- grouping and topology,
- stability across versions where possible.

### **Semantic Roles**
- canonical roles such as:
  - `filter.cutoff`,
  - `filter.resonance`,
  - `osc.tune`,
  - `amp.attack`,
  - `fx.mix`,
  - `global.outputGain`,
- learned through:
  - vendor metadata,
  - naming heuristics,
  - aggregated user actions.

### **Mappings**
- variant → variant mappings for plugin updates,
- format → format mappings for plugin families,
- plugin → plugin semantic mapping for cross-plugin transitions,
- value scaling conversions if parameters use different units.

### **User-Derived Metadata**
- renamings,
- tags,
- plugin categories,
- parameter remappings,
- plugin substitutions.

All collected anonymously and combined probabilistically.

### **Hardware Device Metadata**
- device identity and stable capability fingerprints:
  - vendor, family, model,
  - driver/API type (ASIO/CoreAudio/WASAPI/ALSA/etc.),
  - number of inputs/outputs,
  - typical grouping (stereo pairs, phones outs, ADAT banks),
- common community-preferred role mappings:
  - "outputs 1–2 = main monitors",
  - "outputs 7–8 = headphone out",
  - "ADAT 1–8 = preamp expansion",
- compatible replacements between devices with similar I/O footprints
  (e.g. "Ultralite AVB → Ultralite Mk5 → RME Babyface → Built-In Output"),
- alias templates for hardware devices:
  - e.g. `interface.studioMain`, `interface.laptopDefault`, etc.

Composer does NOT need a full DSP profile; high-level capability knowledge is
sufficient.

### **Control Surface Device Intelligence**

Composer maintains knowledge about control surfaces, MIDI controllers, and assistive hardware:

- **Device fingerprints**: accepts HardwareDevice fingerprints from Pulse containing:
  - vendor/product identifiers,
  - ports and capabilities (keys, pads, faders, encoders, grids, LEDs, displays),
  - sysex identity replies,
  - HID descriptors (summarised).

- **DeviceProfiles**: returns device intelligence including:
  - device class (Keyboard, Pad Controller, MCU Surface, Grid Surface, etc.),
  - default mapping recommendations for different contexts (Arranger, Mixer, Launcher, Plugin UI),
  - known protocol quirks (e.g., special DAW modes, sysex handshakes),
  - LED and display protocols, if known.

- **Statistics and learning**: collects anonymised statistics about:
  - how users override mappings,
  - which mappings are most common,
  - which modes are used most for a given device,
  - uses these statistics to **improve default mappings** over time.

---

## 4. Core Data Model

### 4.1 Plugin Identity

Composer identifies plugins using two layers:

- **Plugin Family**
  A logical grouping representing “this is fundamentally the same plugin across
  formats and versions.”
  Example: `uhe.diva`, `soundtoys.decapitator`.

- **Plugin Variant**
  A specific version + format + platform combination.
  Example: `uhe.diva@vst3_1.4.5`, `uhe.diva@clap_1.4.5`.

Variants include:
- vendor and product names,
- format-specific IDs,
- internal host identifiers,
- fingerprints when determinable.

Pulse uses this to match installed plugins to Composer metadata.

### 4.2 Parameter Identity

Composer stores:

- **Variant-level parameter IDs**:
  the plugin’s native parameter identifiers for a specific variant,

- **Family-level logical parameter IDs**:
  stable, cross-version identifiers that represent the “same control”.

Mappings between variant parameters and logical parameters may originate from:

- vendor integration,
- Composer heuristics,
- user-derived mappings.

### 4.3 Semantic Roles

Semantic roles classify parameters by *meaning*, enabling cross-plugin mapping.

Examples:
- `filter.cutoff`,
- `filter.resonance`,
- `fx.drive`,
- `fx.mix`,
- `eq.lowShelfGain`,
- `amp.attack`,
- `global.outputGain`.

Roles are crucial for automation mapping when replacing plugins.

### 4.4 Plugin Categories and Tags

Composer aggregates:

- plugin categories (EQ, Synth, Compressor, Utility),
- tags (Saturation, Mastering, Drum Bus, Pad Synth, Colour, Surgical),
- inferred metadata (popularity, typical usage).

These suggestions populate Aura’s plugin browser and can accelerate initial
organisation for new users.

### 4.5 User Actions as Signals

Loophole instances may (optionally) report anonymised signals such as:

- parameter renames,
- plugin categorisation assignments,
- plugin substitutions,
- parameter remapping confirmations,
- automation-target mapping decisions.

Composer aggregates these signals statistically to refine metadata and mapping
confidence.

### 4.6 Hardware Device Identity

Composer identifies hardware devices using:

- **Device Family**
  A logical grouping representing devices with similar capabilities and I/O
  footprints.
  Example: `motu.ultralite`, `rme.babyface`, `focusrite.scarlett`.

- **Device Variant**
  A specific model and revision combination.
  Example: `motu.ultralite.mk5`, `motu.ultralite.avb`.

Variants include:
- vendor and product names,
- driver/API type,
- input/output channel counts,
- typical channel labelling,
- known channel groupings (stereo pairs, ADAT banks, etc.).

### 4.7 Hardware Capability Fingerprints

Composer maintains "capability fingerprints" for known devices. Fingerprints
include:

- channel count (inputs and outputs),
- typical channel labelling,
- typical use patterns (based on aggregated usage),
- known quirks (e.g. "channels 7–8 are phones on RME UFX"),
- common role mappings observed in the community.

Fingerprints allow Composer to recommend:
- role mappings when a new device appears,
- fallback mappings when the original device is missing,
- compatible device substitutions.

This is optional and heuristic; Pulse/Aura always retain final control.

---

## 5. Client Integration (Pulse and Aura)

### 5.1 Pulse Knowledge Layer

Pulse includes a thin integration layer which:

- caches Composer responses locally,
- stores local overrides and user-specific metadata,
- exposes a stable API for:
  - plugin descriptions,
  - parameter metadata,
  - mapping suggestions,
  - categories and tags,
  - hardware device profiles,
  - role mapping suggestions,
  - fallback recommendations.

Other Pulse systems never talk to Composer directly.

### 5.2 Interaction with Parameter Aliasing

Pulse's aliasing layer (defined in `10-parameters.md`) uses Composer when:

- local alias mappings are insufficient,
- a plugin changes version,
- a plugin format changes,
- a plugin is replaced with a different plugin.

Pulse merges:

- exact matches,
- alias table matches,
- Composer suggestions (with confidence),
- heuristics,
- user choices.

Pulse always decides the final mapping.

### 5.3 Interaction with Aura

Aura indirectly benefits from Composer through Pulse:

- plugin browser initialisation,
- tag/category suggestions,
- remapping UIs for plugin updates,
- automation lane retargeting suggestions,
- plugin search improvements.

Aura should never depend on live Composer for correctness — cached and local
metadata must always allow a project to load.

### 5.4 Interaction with Hardware I/O

Composer assists Pulse/Aura in determining hardware device mappings, but does
not enforce them. Composer can provide:

- **role → alias** suggestions when a new device appears on a machine,
- **alias → device/channel** default mappings,
- fallback mappings when:
  - a project was authored on a different machine,
  - or with different hardware,
  - or with a more complex I/O box.

Composer provides recommendations based on:
- community data,
- frequency of observed mappings,
- heuristics based on channel count and grouping.

Pulse/Aura always retain final control. Hardware alias→device mappings remain
**local per machine**; Composer's knowledge is global/community-level, not
machine-specific.

Composer only assists in making good guesses:
- when a project is opened on a new machine,
- when hardware changes,
- or when a user plugs in an unfamiliar device.

This separation of concerns ensures that machine-specific hardware
configurations remain local, whilst Composer provides community-driven
recommendations to improve the initial mapping experience.

### 5.5 Interaction with Control Surfaces

Composer provides device intelligence for control surfaces and MIDI controllers:

- **Device fingerprinting**: Pulse sends HardwareDevice fingerprints to Composer when hardware connects. Composer responds with DeviceProfiles containing default mappings and device class information.

- **Default mappings**: Composer provides recommended mapping rules for different contexts (Arranger, Mixer, Launcher, Plugin UI), enabling plug-and-play behaviour.

- **Learning and improvement**: Composer collects anonymised statistics about user mapping overrides and uses this data to improve default mappings over time.

- **Protocol knowledge**: Composer maintains information about device-specific protocols (sysex handshakes, LED protocols, display protocols) to assist Pulse in generating appropriate Feedback Intents.

Pulse remains authoritative for all mapping decisions; Composer provides advisory intelligence to enhance the user experience.

---

## 6. Use Cases

### 6.1 Plugin Version Updates

Pulse identifies a plugin variant change and queries Composer for:

- logical parameter matching,
- suggested mappings with confidence scores,
- renamed or deprecated parameters.

Pulse applies:
- high-confidence mappings automatically,
- ambiguous mappings after user confirmation.

### 6.2 Switching Plugin Formats

Switching, for example, from:

- `MySynth` VST3 → `MySynth` CLAP

Pulse uses Composer’s variant → variant mapping and logical IDs to preserve:

- automation,
- parameter values,
- user settings.

This feature is not offered by any existing DAW.

### 6.3 Plugin Replacement and Meaning-Based Mapping

User replaces a synth with a different synth.

Pulse queries Composer:

- “What are `filter.cutoff`, `filter.resonance`, `amp.attack` equivalents for this new plugin?”

Composer returns candidate parameters with confidence levels.

Pulse remaps automation appropriately.

### 6.4 Plugin Organisation and Browsing

New Loophole install:

- Pulse scans plugins,
- Composer suggests categories/tags,
- Aura presents a curated browser out-of-the-box.

Users can customise freely; their choices are cached and may contribute to
Composer if they opt in.

### 6.5 Hardware Device Recognition and Role Mapping

When a new hardware device is detected:

- Pulse queries Composer for device identity and capability fingerprint,
- Composer suggests common role mappings (e.g. "outputs 1–2 = main monitors"),
- Pulse/Aura present suggestions to the user for confirmation,
- user choices are cached locally and may contribute to Composer if opted in.

### 6.6 Cross-Machine Hardware Fallback

When a project is opened on a different machine:

- Pulse identifies missing hardware devices referenced by device aliases,
- Composer suggests compatible replacements based on capability fingerprints,
- Pulse/Aura present fallback options (e.g. "RME UFX → RME Babyface → Built-In"),
- user confirms or selects alternative mappings,
- mappings are stored locally per machine.

---

## 7. Sync, Caching and Offline Behaviour

Composer integration must be:

- **non-blocking**,
- **cached locally**,
- **optimistic** (refresh-only, not required for operation).

When offline:
- Pulse relies solely on cached and local metadata,
- fallback behaviour (host API + heuristics) always guarantees project safety.

---

## 8. Privacy and Consent

Composer must be opt-in for contributions.

Shared data is strictly limited to:
- plugin/parameter IDs,
- names and tags assigned,
- mapping decisions (anonymous and aggregated),
- plugin substitution and remapping choices,
- device model identifiers,
- channel counts and labels,
- user-applied role mappings (anonymised and aggregated),
- device alias usage patterns,
- fallback mapping choices.

Composer never receives:
- audio,
- arrangement structure,
- file names,
- track content,
- Clip data,
- user identity,
- machine identifiers,
- device serial numbers,
- personally identifiable information.

---

## 8.1 Security & Privacy Boundaries

Composer operates as an external intelligence service with strict data boundaries:

- **Minimum metadata only**: Composer receives only the minimum metadata and features needed to perform its tasks. By default, Composer does **not** receive raw project audio, only fingerprints, analysis summaries, or behavioural signatures (unless explicitly required for a specific AI workflow and explicitly opted-in by the user).

- **Opt-in telemetry**: All telemetry and metadata sharing is **opt-in** and anonymised. Users must explicitly consent before any data is transmitted to Composer.

- **Advisory role**: Composer never acts on the project directly; it only returns suggestions or metadata via Pulse. Pulse remains the authoritative source of truth and validates all Composer suggestions before applying them.

- **Offline operation**: Projects must load and function correctly without Composer. All Composer data is advisory and cached locally; project integrity never depends on Composer availability.

---

## 9. Failure Modes and Fallbacks

If Composer is unreachable or returns errors:

Pulse:
- loads projects using local aliasing and cached metadata,
- maintains full parameter semantics from project files,
- degrades plugin browsing to local metadata only.

Aura:
- simply omits Composer-driven suggestions,
- preserves full editability and control.

Project integrity is always preserved.

---

## 10. Deterministic Behaviour Telemetry and Inference

Composer gathers aggregated telemetry about plugin **determinism vs randomness**
and uses statistical inference to classify nodes as **deterministic-safe** or
**non-deterministic**. This metadata informs Pulse's processing cohort (PC) assignment
decisions, enabling safe anticipative rendering while preserving live performance
for non-deterministic plugins.

### 10.1 What Composer Learns

Composer receives anonymous aggregate telemetry about:

- whether a plugin appears deterministic or not in real projects,
- patterns in outputs when fed identical inputs,
- plugin behaviour under automation sweeps,
- responsiveness to transport seeking,
- stability across versions and formats (CLAP vs VST3 vs AU),
- frequency of cohort switching in Signal triggered by plugin behaviour,
- user overrides marking a node "Force Live" or "Prefer Pre-Render".

Composer does **not** receive any audio — only behavioural signatures. These
signals are aggregated across many Loophole instances to build statistical
confidence about plugin behaviour patterns.

### 10.2 How Composer Interprets Data

Composer uses:

- statistical correlation,
- clustering across similarity classes,
- confidence scoring,
- repeated-observation weighting,
- conflict resolution rules

to determine whether a plugin is **deterministic-safe** or **non-deterministic**.

Composer stores:

- per-plugin-identity classification,
- classification per version,
- classification per format.

Pulse receives:

- `deterministic: true/false`,
- a confidence score (`deterministicConfidence: 0.0–1.0`),
- any known conditions (e.g., “non-deterministic when a UI is open”, “LFOs not
  time-locked”, etc.) as `deterministicConditions: string[]`.

### 10.3 How This Informs Processing Cohorts

Pulse consults Composer metadata when determining:

- which nodes can be moved to the anticipative cohort,
- which must remain live,
- when replacing a plugin with another instance or alternative format,
- when deciding if automation on that plugin is safe for anticipative execution.

Composer never makes decisions itself; it only provides metadata. Pulse combines
Composer's suggestions with local heuristics, user overrides, and real-time
observation to assign nodes to cohorts.

### 10.4 Privacy and Safety Boundaries

Composer:

- receives no project content,
- receives no identifiers,
- receives no audio or MIDI data,
- receives only aggregate behavioural signatures.

All telemetry is anonymised and aggregated before contributing to classification
models. Individual project details are never transmitted or stored.

### 10.5 Relationship to User Overrides

User actions such as:

- “force live”,
- “prefer anticipative”,
- “mark as non-deterministic”,

are logged as *weak telemetry signals* contributing to Composer confidence
scores. These signals are weighted appropriately but do not override statistical
patterns from larger observation sets.

### 10.6 Expected Outputs

Composer exposes the following metadata to Pulse:

- `deterministic: boolean` — whether the plugin is classified as deterministic,
- `deterministicConfidence: 0.0–1.0` — confidence in the classification,
- `deterministicConditions: string[]` — optional conditions affecting
  determinism (e.g., “non-deterministic when UI open”).

Pulse uses these values in cohort assignment logic, combining them with local
observation and user preferences to make final decisions.

### 10.7 Hardware Profile Telemetry

Composer may receive anonymised, opt-in telemetry about hardware devices:

- device model identifiers,
- channel counts and labels,
- user-applied role mappings,
- device alias usage,
- fallback mapping choices.

Composer aggregates this data to improve:
- device recognition accuracy,
- role mapping suggestions,
- fallback recommendations,
- capability fingerprint refinement.

Privacy boundaries:
- No audio data,
- No per-project content,
- No personally identifiable information,
- No machine identifiers,
- Only aggregated, anonymised patterns.

This telemetry follows the same opt-in model as plugin telemetry and is
aggregated statistically to improve community-wide recommendations.

---

## 11. Hardware Knowledge API

Composer exposes conceptual API endpoints (not part of Pulse IPC) for hardware
device knowledge:

- `suggestRoleMappings(deviceDescriptor)` — suggests common role mappings for a
  device based on community patterns,
- `suggestAlias(deviceDescriptor)` — recommends device alias templates (e.g.
  `interface.studioMain`) based on device characteristics,
- `suggestFallbackForRole(roleName, deviceCapabilities)` — suggests compatible
  device/channel mappings when the original device is unavailable,
- `getDeviceProfile(deviceVendorModel)` — returns capability fingerprint and
  known characteristics for a device,
- `matchCapabilities(targetCapabilities)` — finds devices with similar I/O
  footprints and capabilities.

These are conceptual APIs; actual integration occurs through Pulse's knowledge
layer, which caches responses and provides fallback behaviour when Composer is
unavailable.

---

## 12. Future Extensions

Composer will naturally evolve to support:

- vendor-driven metadata ingestion,
- advanced plugin similarity models,
- DSP behavioural clustering (e.g. "similar compressors"),
- cross-plugin preset translation,
- collaborative tag curation,
- improved semantic parsing for parameter naming,
- modulation-role discovery,
- enhanced hardware device compatibility models,
- driver-specific capability detection.

These features all build on Composer's core architecture described here.
