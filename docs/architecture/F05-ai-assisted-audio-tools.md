# AI-Assisted Audio Tools Architecture

This document defines the architecture for **AI-assisted tools** in Loophole.
These tools augment, not replace, the traditional workflow. They focus on:

- suggestions and guidance (never hidden, non-deterministic processing),
- optional one-shot transforms (user-controlled and undoable),
- non-blocking analysis that enhances decision-making,
- tight integration with existing editors and processors,
- Composer-backed intelligence where appropriate.

It complements:

- `06-processing-cohorts-and-anticipative-rendering.md`
- `09-node-graph.md`
- `10-processing-cohorts-and-anticipative-rendering.md`
- `18-midi-architecture.md`
- `19-comping-architecture.md`
- `22-editor-architecture.md`
- `23-diagnostics-and-performance.md`
- `33-plugin-library-and-browser.md`
- Composer architecture (`05-composer.md`)

AI features must never compromise **audio safety**, **project determinism**
(unless explicitly opted into), or **user control**.

---

# 1. Goals

Loophole’s AI-assisted tools aim to:

1. **Enhance, not obscure**
   - Provide visible suggestions and options.
   - Never silently change the mix/arrangement without explicit user action.

2. **Stay non-destructive by default**
   - Use Clips, Pipelines and Nodes to represent changes.
   - All transforms are undoable and inspectable.

3. **Be opt-in**
   - Users choose when and where AI input applies.
   - Default behaviour remains fully manual and deterministic.

4. **Leverage Composer for intelligence**
   - Use Composer’s metadata, classification, and model outputs.
   - Keep Pulse/Signal free from heavy AI execution.

5. **Respect real-time constraints**
   - No heavy inference on the realtime audio thread.
   - All AI workloads run asynchronously and out-of-process.

---

# 2. Roles of Each Layer

## 2.1 Aura

Aura provides:

- UI for invoking AI tools (buttons, menus, overlays).
- Visualisation of suggestions (e.g. automation proposals, comp suggestions).
- Editor overlays (e.g. suggested slice points, harmonies, drum replacements).

Aura never performs heavy AI computation itself. It:

- sends requests to Pulse,
- displays results from Pulse / Composer.

## 2.2 Pulse

Pulse:

- orchestrates AI requests and responses,
- integrates AI-driven suggestions into the project model,
- ensures all AI operations are expressed as:
  - new Clips, Lanes, Nodes,
  - or suggested edits (not applied until approved).

Pulse also:

- maintains AI job queues and statuses,
- ensures deterministic project state where required,
- coordinates with Signal for analysis inputs (e.g. audio stems).

## 2.3 Signal

Signal:

- provides analysis input:
  - prints, feature maps, stems, intermediate buffers,
- runs lightweight analysis DSP (e.g. level/loudness metrics),
- never runs heavy AI inference (models live externally).

## 2.4 Composer

Composer is the primary AI “brain”:

- models for:
  - tagging and classification (clips, samples, plugins),
  - pattern recognition (rhythm, melody, harmony),
  - suggestion generation (EQ shapes, compression, routing),
  - transformation parameters (e.g. vocal clean-up presets).

Pulse → Composer requests are clearly scoped and versioned.

---

# 3. Types of AI-Assisted Tools

This document defines patterns, not a full exhaustive feature list. Example categories:

1. **Analysis & Insight**
   - Intelligent metering views (e.g. spectral “problem zones”).
   - Loudness / balance analysis (e.g. “kick vs bass masking” flags).
   - Clip/track tagging (e.g. “snare”, “rimshot”, “pad”, “staccato strings”).

2. **Editing Assistance**
   - Suggested comp lanes / takes.
   - Suggested timing fixes (warp marker proposals).
   - Suggested slice points in audio (drums, phrases).
   - Clip labelling and segmentation.

3. **Performance / Musical Tools**
   - Groove extraction and suggestion.
   - Harmony / chord voicing suggestions.
   - MIDI pattern continuation.
   - Drum pattern generation based on current kit and style.

4. **Mixing & Processing Helpers**
   - Suggested EQ curves (visual overlays, not enforced).
   - Suggested compression ratios and thresholds.
   - “Before/after” proposals for saturation, widening, de-essing.
   - Smart starting points for plugin chains.

5. **Library & Browser Intelligence**
   - Suggesting plugins for a role (e.g. “gentle bus compressor”).
   - Suggesting samples/clips from the media library.
   - Suggesting plugin chains based on context.

---

# 4. Interaction Model

AI tools follow a consistent interaction model:

1. **User explicitly invokes a tool**
   - e.g. “Analyse this vocal track”, “Suggest a drum pattern for this section”.

2. **Aura → Pulse AI Request**
   - `ai.request` with:
     - scope (clip, track, selection),
     - context (time range, project metadata),
     - intent (analysis vs. proposal vs. transform).

3. **Pulse → Composer / Signal**
   - Gathers needed data:
     - stems, MIDI, feature summaries.
   - Sends a request to Composer or runs lightweight internal logic.

4. **Composer → Pulse Response**
   - Returns:
     - annotations (e.g. markers, tags, suggestions),
     - proposed edits,
     - candidate transformations and their parameters.

5. **Pulse emits “Suggestion” events**
   - `ai.suggestion.*` events to Aura.

6. **Aura displays suggestions**
   - overlay layers,
   - ghost notes/automation,
   - side-by-side “suggested” vs “current” views.

7. **User chooses action**
   - accept,
   - partially apply,
   - reject,
   - tweak and apply.

8. **Pulse applies changes as normal edits**
   - via existing editing primitives (no special “AI mode”).
   - fully undoable, tracked in history.

---

# 5. Data & Determinism

AI-assisted tools must not make Loophole’s behaviour unknowable.

## 5.1 Determinism Constraints

- Offline/bounce renders should be deterministic per **project state**.
- AI models may be non-deterministic internally, but their outputs are:
  - stored in the project (e.g. proposed warp markers, automation),
  - explicitly applied as edits.

Once applied, project state is deterministic.

## 5.2 Snapshotting & Reproducibility

When an AI suggestion is accepted:

- Pulse stores:
  - the resulting edit(s),
  - a minimal reference to how it was generated, for diagnostics:
    - model version, input context hash, etc. (optional).
- On project reload, the AI is not re-run automatically; edits are just normal data.

---

# 6. Example Workflows

## 6.1 AI-Assisted Vocal Cleanup

1. User selects a vocal clip or track segment.
2. Invokes “Analyse vocal for clean-up”.
3. Pulse requests:
   - audio stem from Signal,
   - context from project (genre tags, tempo),
   - optional Composer hints.
4. Composer returns:
   - suggested EQ profile (e.g. rumble cut, harshness notch),
   - suggested de-esser thresholds and ranges,
   - an ordered chain suggestion.
5. Aura shows:
   - overlay nodes on EQ curve,
   - “Apply as new Node chain” option.
6. User accepts:
   - Pulse creates new Nodes in Channel chain,
   - sets parameters accordingly,
   - logs transaction for undo.

## 6.2 Drum Pattern Suggestions

1. User highlights a section and current drum kit Track.
2. Invokes “Suggest a fill” or “Continue this pattern”.
3. Pulse:
   - collects existing MIDI for context,
   - queries Composer for suggested patterns.
4. Composer returns:
   - one or more candidate patterns (MIDI,
     with per-step velocity, probability, etc.).
5. Aura displays:
   - ghost patterns in the Drum Sequencer editor,
   - audition options.
6. User picks a candidate:
   - Pulse commits clip content as normal MIDI.

## 6.3 Clip Labelling & Tagging

1. User runs “Tag this audio library folder” on a set of media.
2. Pulse:
   - requests offline analysis via Composer,
   - receives tags (instrument, mood, genre, spectral character).
3. Tags are applied to media items in the Media Library.
4. Plugin Library / Media Browser search becomes richer over time.

---

# 7. IPC Overview

A dedicated AI domain in Pulse IPC is optional but likely, with patterns such as:

### Aura → Pulse

- `ai.requestAnalysis`
- `ai.requestSuggestion`
- `ai.requestTransformPreview`
- `ai.cancelRequest`

### Pulse → Aura

- `ai.requestQueued`
- `ai.progress` (optional)
- `ai.suggestionReady`
- `ai.error`

Internally, Pulse routes:

- to Composer (network or local service) for heavy AI,
- to Signal for any necessary stems or metrics,
- back to the existing editing primitives to apply or propose changes.

Exact IPC schemas are to be defined once specific AI features are selected for v1.

---

# 8. Performance & Safety

- All AI compute runs:
  - off the audio thread,
  - off the UI thread (for long-running tasks),
  - in separate processes where appropriate.
- AI requests:
  - are cancellable,
  - may degrade gracefully (e.g. low-confidence suggestions, timeouts),
  - must fail safe (no partial project corruption).

Signal is never blocked waiting for AI results.

---

# 9. UX Principles

AI tools must:

- be clearly labelled as suggestions,
- highlight what they will change,
- never silently auto-apply significant edits,
- be easily disabled or ignored,
- integrate with familiar editing contexts (no “black box AI page”).

---

# 10. Future Extensions

Potential future AI-assisted features that fit this architecture:

- Style-matching EQ and dynamics profiles.
- Full mix critique and “fix list” suggestions (human actionable).
- Song structure suggestion from sketch ideas.
- Intelligent crossfade and transition candidates.
- Automatic stem generation for performance use.
- Smart re-amping and tone-matching suggestions.

All of these remain grounded in explicit user action and visible edits.

---

# 11. Summary

The AI-Assisted Audio Tools architecture:

- treats AI as a **suggestive layer** over core editing,
- ensures all changes are explicit, visible and undoable,
- relies on Pulse for orchestration and Composer for heavy intelligence,
- integrates seamlessly with existing editors and the NodeGraph,
- and maintains Loophole’s commitment to deterministic, project-based state.

AI in Loophole is designed to amplify creativity and speed, never to hide control
from the user.
