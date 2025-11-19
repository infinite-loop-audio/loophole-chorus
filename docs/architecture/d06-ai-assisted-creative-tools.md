# AI-Assisted Creative Tools Architecture

## Contents

- [1. Overview](#1-overview)
- [2. Design Philosophy](#2-design-philosophy)
- [3. Architectural Responsibilities](#3-architectural-responsibilities)
  - [3.1 Aura](#31-aura)
  - [3.2 Pulse](#32-pulse)
  - [3.3 Composer](#33-composer)
  - [3.4 Signal](#34-signal)
- [4. Semantic Foundation](#4-semantic-foundation)
- [5. Core Assistive Features](#5-core-assistive-features)
  - [5.1 Intelligent Kit Builder](#51-intelligent-kit-builder)
  - [5.2 StackNode Variant Assistant](#52-stacknode-variant-assistant)
  - [5.3 Clip and Scene Variation Engine](#53-clip-and-scene-variation-engine)
  - [5.4 Media & Plugin Browser Copilot](#54-media--plugin-browser-copilot)
  - [5.5 Routing & Node-Chain Assistant](#55-routing--node-chain-assistant)
- [6. Interaction Model](#6-interaction-model)
- [7. Data Flow](#7-data-flow)
- [8. User Experience Principles](#8-user-experience-principles)
- [9. Security & Privacy](#9-security--privacy)
- [10. Constraints](#10-constraints)
- [11. Future Extensions](#11-future-extensions)

---

## 1. Overview

Loophole includes a suite of **AI-assisted creative tools** designed to extend
the musician’s creative intuition without replacing it. These tools operate at a
semantic level using Composer’s metadata models to provide intelligent,
context-aware suggestions, variations, and sound design scaffolding.

AI assistance in Loophole:

- reorganises and augments **existing user-owned material**,  
- makes large libraries feel manageable and expressive,
- provides inspiration without overriding user intent,
- always remains **explicit, reversible, and non-destructive**.

AI is not used for direct audio generation. Instead, it aids users by selecting,
combining, transforming, and arranging material already present in the project
or the user’s library.

---

## 2. Design Philosophy

The creative AI system is founded on five principles:

1. **Human-first creativity**  
   Suggestions are prompts, not prescriptions. The user always chooses the
   result.

2. **No secret operations**  
   AI never modifies the project silently. Every transformation is explicit and
   previewable.

3. **Grounded in the user’s environment**  
   All results come from:
   - the user’s media,
   - their plugins,
   - their presets,
   - their project context.

4. **Semantic, not generative**  
   AI interprets musical descriptions, patterns, and structures, and uses
   deterministic transformations to create variations or proposals.

5. **Transparent and explainable**  
   Users can ask why something was suggested. AI responses aim to teach, not
   obscure knowledge.

---

## 3. Architectural Responsibilities

### 3.1 Aura

Aura handles:

- the interactive UI panels for suggestions, kits, variations, etc.
- collecting contextual selections (clips, tracks, scenes, browser entries),
- presenting Composer’s suggestions with previewing,
- allowing users to apply, audition, or dismiss proposals.

Aura never performs transformation logic directly.

### 3.2 Pulse

Pulse is responsible for **execution**:

- creating new clips/scenes/tracks based on proposals,
- instantiating Node chains,
- inserting StackNode variants,
- updating parameters and automation non-destructively,
- ensuring changes integrate with undo/redo.

Pulse enforces model correctness and safety.

### 3.3 Composer

Composer is the **semantic brain**:

- indexes and tags user media (samples, loops),
- analyses presets, plugin behaviours, Node chains,
- stores embeddings, descriptors, and musical metadata,
- performs contextual searches (“dark piano pads with soft attacks”),
- communicates with LLMs to interpret natural-language queries,
- ranks and returns concrete IDs for Pulse to apply.

### 3.4 Signal

Signal is affected only indirectly:

- it receives new Node chains or StackNodes created by Pulse,
- no AI logic runs inside Signal,
- AI decisions never bypass deterministic DSP rules.

---

## 4. Semantic Foundation

Composer maintains semantic representations for:

- **Samples & loops**  
  (key, tempo, timbre, articulation, tags)

- **Instruments & presets**  
  (“airy pad”, “dark FM bell”, “thick Reese bass”)

- **Effect chains & processors**  
  (compressors that “glue”, delays that “widen”, saturators that “warm”)

- **Musical patterns**  
  (drum grooves, bass movement types, melodic motifs)

- **Track & scene context**  
  (energy, density, arrangement role)

These descriptors allow AI to reason symbolically rather than blindly.

---

## 5. Core Assistive Features

### 5.1 Intelligent Kit Builder

Given a Track, Lane, or Scene, Composer can propose:

- drum kits,
- playable sound sets,
- loop packs,
- sample clusters,
- multi-instrument kits.

The user can specify high-level intent:

- “tight and dry”  
- “lo-fi crunchy”  
- “bright, modern trap hats”  
- “cinematic big percussion”

Pulse creates kits as Tracks or Lane bundles.

### 5.2 StackNode Variant Assistant

When hovering a StackNode, users may request:

- “Suggest alternatives”
- “Suggest warm vocal compressors”
- “Suggest creative distortion chains”

Composer identifies:

- semantically similar plugins,
- style-appropriate chains,
- presets with matching tone profiles.

Pulse inserts variants as new entries in the StackNode.

### 5.3 Clip and Scene Variation Engine

Users can request variations of:

- MIDI clips,
- audio slices (non-destructive),
- entire scenes.

Example prompts:

- “More energy”  
- “Sparser groove”  
- “Double-time hats”  
- “Simplify chord movement”  
- “Turn this into a call-and-response pattern”

Pulse applies **deterministic transformations**, producing a new clip or scene
while preserving the original.

### 5.4 Media & Plugin Browser Copilot

Natural-language search for:

- samples (“soft woody rimshot”, “hyper-compressed old-school snare”),
- presets (“lush pad with slow attack and airy top”),
- plugins (“distortion good for 808s”).

Composer interprets queries and returns:

- ranked lists,
- preview data,
- semantic summaries (“this compressor tightens low-end transients”).

### 5.5 Routing & Node-Chain Assistant

AI can propose:

- parallel chains,
- NY compression,
- widening stacks,
- creative FX chains for sound design.

Commands like:

- “Give this vocal warmth and presence”
- “Add width to this synth”
- “Make a parallel saturated bus”

Pulse instantiates Node graphs accordingly.

---

## 6. Interaction Model

AI actions follow a consistent lifecycle:

1. **User selects context**  
   (clip, track, node, stack, browser entry)

2. **User requests AI assistance**  
   via a small Assist button or command.

3. **Aura gathers context**  
   (musical, structural, semantic, selection state)

4. **Aura sends a structured query to Composer**

5. **Composer returns proposals**  
   (assets, chains, clips, variants + text descriptions)

6. **Aura previews proposals**

7. **User accepts** (or discards)  
   Pulse applies changes in a non-destructive way.

---

## 7. Data Flow

High-level pipeline:

```
Aura → Composer → Aura → Pulse → Signal
```

Breakdown:

- Aura: query builder, previewing, acceptance
- Composer: semantic search & AI interpretation
- Aura: user-facing proposal selection
- Pulse: graph + track + clip changes
- Signal: processes new graph

---

## 8. User Experience Principles

- **Non-destructive by default**  
  All AI edits create new clips, lanes, or StackNode variants.

- **Light-touch UI integration**  
  Assist buttons appear next to interactive areas but never distract.

- **Explainability**  
  Composer may annotate suggestions:
  - “Chosen for its warm midrange”
  - “Pairs well with your existing snare chain”

- **Respect for the musician’s taste**  
  AI does not override user choices or reorder their graph without permission.

- **High-velocity exploration**  
  Rapid A/B switching, multi-variant previews, scene auditioning.

---

## 9. Security & Privacy

- No user media is uploaded unless explicitly permitted.
- All local-only mode features work without cloud access.
- Natural-language queries in offline mode are routed through bundled models.

---

## 10. Constraints

- AI cannot directly generate audio.  
  All transformations must operate on existing content.
- AI never performs irreversible operations.  
  The user must always be able to revert.
- Suggestions must remain musically plausible and context-aware.

---

## 11. Future Extensions

- Style transfer MIDI (non-audio)
- Multi-track groove extraction
- Intelligent “fill generation” for drums
- Scene-based arrangement skeleton suggestions
- Hybrid Composer + cloud models for deeper understanding
