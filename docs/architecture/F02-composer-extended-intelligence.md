# Composer Extended Intelligence Architecture

This document defines the **extended architecture of Composer**, Loophole’s
metadata-driven intelligence and machine-learning service. It expands on the
baseline Composer spec to describe Composer’s *full* future role:

- global knowledge aggregation,
- media and plugin intelligence,
- user-behaviour driven modelling,
- project-context inference,
- AI-assisted suggestions,
- device intelligence,
- semantic search,
- cross-version/plugin mapping,
- deterministic heuristics,
- future neural model integration.

Composer is not a DAW component itself — it is a **shared intelligence layer**
that enriches Pulse and Aura without ever touching Signal or real-time DSP
paths.

It integrates with:

- `05-composer.md` (base architecture)
- `33-plugin-library-and-browser.md`
- `13-media-architecture.md`
- `30-ai-assisted-audio-tools.md`
- `10-processing-cohorts-and-anticipative-rendering.md`
- Hardware & device architecture
- Project metadata (`21-project-metadata.md`)

This extended architecture is deliberately forward-looking and provides a
framework for large-scale intelligence features without forcing them into v1.

---

# 1. Goals

Composer’s extended role is to:

1. **Learn**  
   Acquire anonymised, aggregate knowledge from users’ interactions.

2. **Normalise**  
   Clean, unify, and standardise plugin, media, device and parameter metadata.

3. **Predict**  
   Provide suggestions that respect Loophole’s philosophy of *visible,
   editable, optional* assistance.

4. **Map**  
   Resolve plugin replacements, parameter mapping, and cross-version transitions.

5. **Classify**  
   Tag media, classify plugin roles, infer deterministic vs
   non-deterministic behaviour.

6. **Contextualise**  
   Provide project-aware intelligence:
   - “Suggest a mastering chain”
   - “Find similar samples”
   - “Recommend plugins that suit this track’s role”

7. **Never block workflows**  
   All Composer requests:
   - are asynchronous  
   - are advisory  
   - never break determinism  
   - fail silent and safe  

Composer never performs real-time tasks.

---

# 2. Architectural Overview

Composer is a **stateless API facade** over multiple functional subsystems:

- Metadata Normalisation  
- Heuristic Inference  
- ML/AI Models  
- Recommendation Engine  
- Search & Semantic Embeddings  
- User-Specific Profiles (local)  
- Global Knowledge Pool (cloud opt-in)

Pulse communicates with Composer through a clean, versioned API.  
Aura only indirectly interacts through Pulse.

Signal has **no dependency** on Composer.

---

# 3. Major Functional Domains

## 3.1 Plugin Intelligence

Composer maintains robust intelligence for plugins:

### **3.1.1 Role Classification**
- EQ  
- Compressor  
- Limiter  
- Saturator  
- Delay  
- Reverb  
- Modulator  
- Dynamics utility  
- Multi-effect  
- Instrument types (FM, subtractive, wavetable, sampler-like)

### **3.1.2 Determinism Classification**
Based on:

- Time variance tests  
- Input invariance tests  
- Reported plugin metadata  
- Observed user behaviour (e.g. “this plugin always lands in realtime cohort”)

Outputs a confidence score:

```
PluginDeterminism {
  pluginId;
  deterministic: boolean | "likely" | "unlikely";
  confidence: number;      // 0–1
  evidence: string[];
}
```

### **3.1.3 Variant & Version Mapping**
Composer detects equivalencies between:

- CLAP and VST3 versions  
- Updated versions of the same plugin  
- Legacy → modern plugin evolutions  

Provides replacement suggestions:

```
PluginReplacementSuggestion {
  fromPluginId;
  toPluginId;
  similarity: number;   // 0–1
  mappedParameters: Mapping[];
  confidence: number;
}
```

### **3.1.4 Parameter Aliasing**
Maps parameter names and roles across versions:

```
Cutoff ↔ Frequency
Panorama ↔ Balance
Character ↔ Warmth
```

Useful for:

- plugin replacement,
- preset migration,
- searching by parameter semantics.

### **3.1.5 Smart Defaults**
Given context, Composer can suggest:

- suitable plugin orderings,
- default settings for common roles,
- “mix-ready” starting points,
- templates for chain creation.

All optional.

---

## 3.2 Media Intelligence

Composer helps organise and augment media:

### **3.2.1 Auto-Tagging**
For samples, clips, recordings:

- instrument type  
- timbre (bright, dark, warm, airy)  
- genre/style  
- loop characteristics  
- one-shot vs sustained  
- tempo/key estimation  
- spectral profile class

### **3.2.2 Similarity Search**
Using embeddings:

- “Find me samples like this snare”
- “Find pads similar to this clip”
- “Locate loops that match this groove”

### **3.2.3 Cross-Media Relations**
Composer connects:

- audio↔MIDI inference (“convert melody to MIDI”)
- clip→instrument suggestions
- sample selection for kits or generative instruments

### **3.2.4 Library-Level Recommendations**
Given a Track:

- suggest samples that fit genre,
- suggest one-shot replacements,
- recommend starting points for drum kits.

---

## 3.3 Device Intelligence

Composer maintains a database of:

- MIDI controllers,  
- control surfaces,  
- audio interfaces,  
- pad/grid controllers,  
- fader banks,  
- hybrid devices (Launchpads, Push, Maschine family).

It provides:

- device templates,
- automatic mapping heuristics,
- prime functions per device,
- suggested CC/default mappings,
- mapping conflict resolution.

Pulse queries Composer when new hardware appears.

---

## 3.4 Project Intelligence

Composer can return higher-level project analysis:

### **3.4.1 Structure & Patterns**
- song sections (verse/chorus/bridge)
- density graphs  
- “busy vs sparse” analysis  
- tension curves  
- groove regularity  
- repeating motif identification  

### **3.4.2 Mix Insights**
Non-destructive suggestions:

- “Kick and bass masking around 60Hz”
- “Vocal sibilance unusually high”
- “High-frequency harshness detected in FX bus”

### **3.4.3 Creative Suggestions (Optional)**
- chord progressions that match the piece,
- rhythmic motifs,
- variation ideas,
- “fit this clip into the current key”.

Composer’s suggestions never apply automatically — they are displayed in Aura as
optional, user-reviewable annotations.

---

## 3.5 Semantic Search

Composer exposes semantic search over:

- plugins,
- samples,
- clips,
- presets,
- chains,
- instrument patches.

Example queries:

- “Warm tape-style saturation”
- “Percussive plucks”
- “Moody pads”
- “Clean vocal EQ”
- “Minimal techno drum one-shots”

Pulse normalises the query, Composer generates candidate results, and Aura displays them.

---

# 4. Data Flows

## 4.1 Pulse → Composer

Pulse sends:

- plugin metadata (as needed)
- media analysis requests
- search queries
- plugin similarity queries
- replacement mapping hints
- device identification profiles
- project summaries for context-aware suggestions

All requests are asynchronous and cancellable.

---

## 4.2 Composer → Pulse

Composer returns:

- metadata refinements,
- tag suggestions,
- replacement mappings,
- classification info,
- embeddings (optional opaque vectors),
- suggestions for samples/plugins/clips/chains.

Pulse integrates results into:

- Plugin Library,
- Media Library,
- Editor overlays,
- AI-assisted features.

---

# 5. Privacy & Data Policy

Composer is built on **privacy-first** constraints:

### Opt-in only
Users choose whether to send:

- plugin usage stats,
- categorisation decisions,
- parameter alias data,
- media tagging corrections.

### No raw audio uploads by default
Pulse sends only:

- fingerprints,
- statistics,
- features,
- opt-in content for specific AI workflows.

### All sharing is anonymous
Composer receives **no** personally identifiable information.

### Local versus Global
Each Composer installation should include:

- Local knowledge cache (offline operation),
- Optional global sync layer.

---

# 6. Versioning & Contracts

Composer’s APIs are versioned:

```
composer.apiVersion = "2.0.0"
```

Pulse can:

- query capabilities,
- fallback to simpler behaviour,
- disable advanced suggestions gracefully.

This ensures compatibility across versions and user configurations.

---

# 7. Integration with Other Loophole Subsystems

## 7.1 Plugin Library
Composer enhances:

- categorisation,
- search,
- similarity,
- tag inference,
- variant selection.

## 7.2 Media Library
Composer performs:

- auto-tagging,
- similarity search,
- content classification.

## 7.3 AI-Assisted Tools
Composer is the computation backend.  
Pulse orchestrates workflow; Aura displays results.

## 7.4 Cohorts / Determinism
Composer provides probabilistic classification used when assigning Nodes to realtime or anticipative cohorts.

## 7.5 Hardware & Control Surfaces
Composer offers device templates and mapping intelligence.

---

# 8. Future Capabilities

### 8.1 Adaptive Instrument Models
Generate playable instrument mappings or adjustments per sample library.

### 8.2 Neural Audio Tools
Spectral transformations, style-transfer processing suggestions (never realtime).

### 8.3 Global Network Effects
Plugin popularity/suitability based on real usage patterns.

### 8.4 Project-Level Style Emulation
Composer identifies stylistic fingerprints and suggests fits.

### 8.5 AI-Assisted Editing Negotiation
Composer ranks candidate edits for comping, quantisation, phrasing.

---

# 9. Summary

Composer’s Extended Intelligence Architecture positions Composer as:

- a unifying metadata intelligence layer,
- a semantic engine for search and classification,
- a replacement-mapping and version-migration system,
- an advisory system for mixing, arranging and editing,
- a knowledge engine for plugin, media, device and parameter domains,
- a backend for AI-assisted workflows,
- and a context-aware recommender for the whole DAW.

It enhances Loophole without affecting real-time safety or determinism, and
provides a long-term foundation for deeper AI and metadata-driven features.
