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
- [5. Client Integration (Pulse and Aura)](#5-client-integration-pulse-and-aura)
  - [5.1 Pulse Knowledge Layer](#51-pulse-knowledge-layer)
  - [5.2 Interaction with Parameter Aliasing](#52-interaction-with-parameter-aliasing)
  - [5.3 Interaction with Aura](#53-interaction-with-aura)
- [6. Use Cases](#6-use-cases)
  - [6.1 Plugin Version Updates](#61-plugin-version-updates)
  - [6.2 Switching Plugin Formats](#62-switching-plugin-formats)
  - [6.3 Plugin Replacement and Meaning-Based Mapping](#63-plugin-replacement-and-meaning-based-mapping)
  - [6.4 Plugin Organisation and Browsing](#64-plugin-organisation-and-browsing)
- [7. Sync, Caching and Offline Behaviour](#7-sync-caching-and-offline-behaviour)
- [8. Privacy and Consent](#8-privacy-and-consent)
- [9. Failure Modes and Fallbacks](#9-failure-modes-and-fallbacks)
- [10. Future Extensions](#10-future-extensions)

---

## 1. Overview

Composer is Loophole’s shared knowledge metadata service. It maintains a
continuously improving model of:

- plugins (their identities, formats and versions),
- parameters (their names, types, ranges, roles and groupings),
- relationships between plugins (mapping across versions and formats),
- semantic equivalences across different plugins,
- plugin categories and user-driven classification.

Composer is optional but enhances Loophole’s resilience and ergonomics. When
available, it supports:

- smarter automation remapping,
- robust project loading after plugin updates,
- automatic tagging and organisation of plugin collections,
- semantic mapping when replacing plugins,
- better UX in browsing, categorisation and preset behaviour.

Composer suggestions are non-authoritative. Pulse combines them with local
aliasing, heuristics and user preferences to choose the final mapping.

---

## 2. Goals

Composer aims to:

- prevent loss of automation and parameter state when plugins change,
- reduce friction in plugin organisation and discovery,
- allow semantic operations across plugins (e.g. “map cutoff to cutoff”),
- provide a shared ecosystem of metadata continuously improved by user input,
- support long-term sustainability for projects spanning multiple plugin
  versions and formats.

Composer must:

- be optional and non-blocking,
- have no access to project media or arrangement data,
- work entirely from plugin and parameter metadata,
- protect privacy and operate on anonymised signals,
- function gracefully offline with local caching.

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
  - categories and tags.

Other Pulse systems never talk to Composer directly.

### 5.2 Interaction with Parameter Aliasing

Pulse’s aliasing layer (defined in `04-parameters.md`) uses Composer when:

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
- plugin substitution and remapping choices.

Composer never receives:
- audio,
- arrangement structure,
- file names,
- track content,
- Clip data,
- user identity.

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

## 10. Future Extensions

Composer will naturally evolve to support:

- vendor-driven metadata ingestion,
- advanced plugin similarity models,
- DSP behavioural clustering (e.g. “similar compressors”),
- cross-plugin preset translation,
- collaborative tag curation,
- improved semantic parsing for parameter naming,
- modulation-role discovery.

These features all build on Composer’s core architecture described here.
