# Composer Semantic Architecture

## Contents

- [1. Overview](#1-overview)
- [2. Role of Composer in the Loophole Ecosystem](#2-role-of-composer-in-the-loophole-ecosystem)
- [3. Semantic Data Model](#3-semantic-data-model)
  - [3.1 Assets](#31-assets)
  - [3.2 Descriptors & Embeddings](#32-descriptors--embeddings)
  - [3.3 Context Models](#33-context-models)
  - [3.4 Provenance & Confidence Scoring](#34-provenance--confidence-scoring)
- [4. Semantic Processing Pipeline](#4-semantic-processing-pipeline)
- [5. Natural-Language Query Interpretation](#5-natural-language-query-interpretation)
- [6. Suggestion Engines](#6-suggestion-engines)
  - [6.1 Sound & Sample Suggestion](#61-sound--sample-suggestion)
  - [6.2 Plugin & Processor Suggestion](#62-plugin--processor-suggestion)
  - [6.3 Chain & Routing Suggestion](#63-chain--routing-suggestion)
  - [6.4 Variation & Pattern Suggestion](#64-variation--pattern-suggestion)
- [7. Interaction with Pulse](#7-interaction-with-pulse)
- [8. Interaction with Aura](#8-interaction-with-aura)
- [9. Interaction with Signal](#9-interaction-with-signal)
- [10. Data Storage & Locality](#10-data-storage--locality)
- [11. Privacy & Online Operation](#11-privacy--online-operation)
- [12. Future Extensions](#12-future-extensions)

---

## 1. Overview

Composer is the **semantic intelligence layer** of Loophole.

It is responsible for:
- understanding what audio, MIDI, plugins, presets, and Node chains *are*,
- organising these items into a semantic graph,
- interpreting natural-language queries,
- answering musical questions in context,
- generating suggestions for Pulse to apply in the project.

Composer is **not**:
- a DSP engine,
- a generative audio model,
- a rules-only system.

Instead, it blends metadata, feature extraction, context modelling, and LLM-based natural-language interpretation to create a unified musical intelligence layer.

---

## 2. Role of Composer in the Loophole Ecosystem

Composer sits between the user-facing UI (Aura) and the structural engine (Pulse):

- Aura → *“I need kits that fit this chorus.”*  
- Composer → *“Here are kits sorted by relevance, with explanations.”*  
- Pulse → *“I will apply the selected kit as Nodes, tracks, or clips.”*  
- Signal → unaware of AI; executes whatever Pulse gives it.

Composer is responsible for:
- semantic search,
- semantic clustering,
- similarity scoring,
- autofill of metadata,
- style/instrument classification,
- behavioural classification of plugins and chains,
- feature extraction from audio and MIDI.

---

## 3. Semantic Data Model

### 3.1 Assets

Composer maintains semantic records for:

**Samples & Loops**
- features: spectral centroid, transient pattern, articulation, sibilance
- metadata: tempo, key, tuning, loopability, start-point safety
- tags (user & detected)

**Instruments & Presets**
- synthesis method (FM, subtractive, wavetable…)
- timbral profile (bright, dark, airy, woody)
- dynamic profile (soft attack, percussive, evolving)

**Plugins & Processors**
- category (comp, EQ, saturator)
- behaviour model:  
  - transient-tightening, warming, widening, compressive, clean, aggressive
- derived properties (oversampling, linear-phase, stereo behaviour)

**Node Chains**
- combined effects
- routing topology
- musical role (glue, colour, width, punch)

**Clips**
- complexity, density
- rhythmic DNA
- motif classification
- groove signatures

**Scenes & Sections**
- energy level
- harmonic openness
- orchestration footprint

---

### 3.2 Descriptors & Embeddings

Composer uses multiple representation layers:

- **Symbolic descriptors**  
  Human-readable tags and attributes.

- **Learned embeddings**  
  Multidimensional numeric vectors representing similarity.

- **Hybrid descriptors**  
  E.g. “bright pluck with soft transient”, mapped into embedding space.

Embeddings are:
- deterministic where possible (audio features, MIDI features),
- statistically aggregated where necessary (plugin behaviour),
- refined with Composer’s federated feedback learning.

---

### 3.3 Context Models

Composer builds context models for:

- *Local context*: selected clip/track/scene.
- *Project context*: instrumentation, tempo, density.
- *User taste model*: items repeatedly chosen or rejected.
- *Chain context*: plugins commonly used together.
- *Genre/feel hints*: if detectable (optional, inferred softly).

---

### 3.4 Provenance & Confidence Scoring

Every semantic fact in Composer carries a **provenance trail**:
- user annotation,
- automated feature detection,
- learned cluster membership,
- externally-supplied metadata.

Each carries a **confidence score**, and Composer may produce suggestions like:

> “High confidence: this preset produces soft, plucky textures suitable for intros.”

---

## 4. Semantic Processing Pipeline

Composer’s semantic workflow:

1. **Asset ingestion**
   - new samples, plugins, presets, projects
   - derive technical metadata

2. **Feature extraction**
   - FFT, MFCCs, transient analysis, harmonic profile
   - MIDI structure analysis
   - plugin DSP behaviour probing (non-invasive)

3. **Descriptor generation**
   - tags, category inference
   - similarity computations

4. **Embedding generation**
   - map items to embedding space using local models
   - optionally enhance with cloud model when allowed

5. **Indexing & linking**
   - maintain semantic graph of items, tags, behaviours, roles

6. **Query interpretation**
   - LLM interprets user intent
   - Composer executes structured version

7. **Suggestion ranking**
   - match embeddings
   - apply project/user context
   - produce explanations & confidence

---

## 5. Natural-Language Query Interpretation

Composer handles queries like:

- *“Find a dark pluck for this lead.”*
- *“Suggest a warm vocal compressor alternative.”*
- *“Give me groovier hats for this pattern.”*
- *“What plugins would work well before this tape saturator?”*

Flow:

1. Aura captures query.
2. Aura provides project context + selection context.
3. Composer:
   - normalises query,
   - extracts constraints,
   - interprets descriptors,
   - constructs a structured request.
4. Composer resolves this into asset IDs + explanations.
5. Aura displays them.

---

## 6. Suggestion Engines

### 6.1 Sound & Sample Suggestion
- clustering based on timbral and transient similarity,
- browser integration,
- context-aware auditioning.

### 6.2 Plugin & Processor Suggestion
- match plugin roles to context,
- propose replacements for missing plugins,
- power A/B workflows via StackNodes.

### 6.3 Chain & Routing Suggestion
- propose Node chains for:
  - widening,
  - warming,
  - distortion,
  - drum glue,
  - ambience,
  - colour.

### 6.4 Variation & Pattern Suggestion
- MIDI groove variations,
- phrase transformations,
- scene-level recomposition,
- density adjustments.

---

## 7. Interaction with Pulse

Composer never edits data directly.

Instead, it returns:

- asset IDs,  
- preset IDs,  
- NodeGraph templates,  
- clip transformations,  
- parameter suggestions,  
- scene variants,  
- chain suggestions.

Pulse then:
- instantiates objects,
- ensures model consistency,
- integrates undo/redo,
- sends changes to Signal.

---

## 8. Interaction with Aura

Aura is responsible for:

- displaying suggestions,
- allowing previewing,
- presenting alternative variants,
- contextual search panels,
- invoking Composer via API.

Aura → Composer communication includes:
- selected project region,
- browser state,
- Node chain selection,
- query text,
- user intent (“suggest”, “explain”, “compare”).

---

## 9. Interaction with Signal

Signal does **not** run AI logic.

Composer only affects Signal indirectly through Pulse:
- new Node chains,
- StackNode variants,
- parameter suggestions,
- routing changes.

Signal remains deterministic.

---

## 10. Data Storage & Locality

Composer stores:

- local semantic DB (SQLite + vector DB),
- feature caches,
- model embeddings.

Local-first, with optional:
- cloud model enhancement,
- federated learning (if enabled),
- cloud backup of non-personalised indexes.

---

## 11. Privacy & Online Operation

- User audio, MIDI, and plugin data never leaves the machine unless explicit.
- Query text may be sent to cloud models *only with permission*.
- Offline mode uses bundled models and reduced descriptor sets.
- Semantic results persist across sessions but are isolated per-user.

---

## 12. Future Extensions

- Cross-project motif discovery,
- Style similarity transfer,
- High-level assistant (“explain why this chain works”),
- Multi-turn semantic exploration sessions,
- User taste embedding for personalised suggestions,
- Composer-side plugin parameter aliasing and mapping intelligence.
