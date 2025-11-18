# Implementation Workflow & Tooling

This document defines how Loophole is **actually built**:

- how architecture documents in **Chorus** translate into code in  
  **Signal, Pulse, Aura and Composer**,
- how the **human developer**, **ChatGPT** and **Cursor** interact,
- how changes move from “idea” → “spec” → “implementation” → “tests”,
- and the conventions that keep the project coherent over time.

It is the **engineering companion** to the rest of the architecture set.

---

## 1. Goals & Principles

The implementation workflow is based on a few core principles:

1. **Architecture first**  
   No significant features are implemented without at least a lightweight
   architecture/spec section in Chorus.

2. **Single source of truth**  
   Chorus drives design. The other repos (Signal, Pulse, Aura, Composer) follow.

3. **Clear roles for tools**  
   Human, ChatGPT and Cursor each have well-defined responsibilities and
   expectations.

4. **Documentation is live, not legacy**  
   Specs, IPC and decisions are kept up to date as code evolves.

5. **Safety & determinism first**  
   Testing and validation are part of the workflow, not an afterthought.

---

## 2. Repositories & Responsibilities

Loophole consists of five primary repositories:

- **Chorus** — `loophole-chorus`  
  Architecture, decisions, specs, IPC, backlog, meta docs.

- **Signal** — `loophole-signal`  
  C++/JUCE audio engine, node graph, plugin hosting, hardware IO.

- **Pulse** — `loophole-pulse`  
  Rust model server, project state, editing, IPC orchestration.

- **Aura** — `loophole-aura`  
  Electron/TypeScript UI, editors, media browser, clip launcher, mixer UI.

- **Composer** — `loophole-composer`  
  Metadata + intelligence service, plugin/media/device intelligence.

Implementation flows **from Chorus outwards**:

1. Architecture/spec agreed in Chorus.  
2. IPC (if needed) updated in Chorus.  
3. Code implemented in one or more of Signal / Pulse / Aura / Composer.

---

## 3. Actors & Their Roles

### 3.1 Human Developer

You are the **product owner** and **lead engineer**. You:

- define priorities and direction,
- write and refine architecture docs,
- review generated code and specs,
- run and test the software,
- make final decisions when trade-offs appear.

### 3.2 ChatGPT (Architect & Reviewer)

Within this project, ChatGPT acts primarily as:

- **Architect / Designer**
  - helps shape architecture docs in Chorus,
  - identifies gaps/inconsistencies,
  - drafts spec/IPC/decision documents.

- **Reviewer**
  - suggests refactors,
  - checks coherence between repos and docs,
  - proposes test strategies.

ChatGPT does **not** directly modify files; it produces **content and prompts**
that you feed into Cursor (or edit by hand).

### 3.3 Cursor (Implementation Agent)

Cursor is:

- the **hands** that edit code and docs,
- the primary tool for:
  - applying prompts to architecture docs,
  - writing and refactoring code,
  - running tests locally (via your editor).

Cursor:

- works within a single repo at a time,
- must respect repository-specific conventions (see below),
- must avoid touching old reports unless explicitly told otherwise.

---

## 4. Branching & Git Workflow

The recommended branching model for all Loophole repos:

- `main`
  - release / stable branch,
  - tagged for public releases.

- `develop`
  - default working branch,
  - accumulates features and fixes.

- Feature branches
  - `feature/<short-name>`
  - created from `develop`,
  - merged back via PR or equivalent review.

Basic flow:

1. Branch from `develop` in the relevant repo(s).
2. Implement feature guided by Chorus docs.
3. Run tests and manual checks.
4. Merge into `develop` once stable.
5. `develop` is occasionally merged to `main` as release milestones.

Chorus generally follows the same pattern, but changes there can be more frequent
and smaller.

---

## 5. Documentation & File Conventions

### 5.1 Architecture & Spec Docs

- Located in `docs/architecture/` (Chorus).
- Named `NN-topic-name.md` where `NN` is a stable index number.
- Indexed in `docs/architecture/00-index.md`.
- Each file has:
  - a clear purpose,
  - headings for responsibilities, interactions, and constraints.

**Renumbering** is rare and handled via a dedicated prompt + report.

### 5.2 IPC Specs

- Located in `docs/specs/ipc/` (Chorus).
- Separate trees for:
  - `pulse/`
  - `signal/`
- Each domain/file describes:
  - commands,
  - events,
  - payloads,
  - semantics and safety notes.

### 5.3 Meta & Reports

- `docs/meta/` holds:
  - inbox/backlog,
  - AI workflow docs,
  - protocol guidelines.

- `docs/reports/` holds:
  - machine-generated reports,
  - renumbering logs,
  - cohesion analyses,
  - refinement reports.

**Reports are immutable logs**:

- They must not be edited or deleted,
- New reports use filenames of the form:

  - `YYYY-MM-DD-HHMMSS-descriptive-name.md`

Cursor prompts should always include:

> “Do not modify any existing files under `docs/reports/`; only add a new report.”

### 5.4 Quad-Backtick Wrapping

When ChatGPT produces a full file for you to copy into a repo, it is wrapped in:

- **quad backticks** around the whole file:

````markdown
# File content…
````

This prevents nested triple-backtick code blocks from breaking.

---

## 6. Change Types & Recommended Workflow

Not all changes follow the same path. There are three main categories:

### 6.1 Architecture-First Feature

Use this for **new features** or **non-trivial changes**.

1. **Idea capture**
 - Add high-level idea to `architecture-backlog.md` (Chorus meta),
   or to `architecture-inbox.md` if still fuzzy.

2. **Architecture doc**
 - Either:
   - extend an existing doc, or
   - add a new `NN-topic.md` in `docs/architecture/`.
 - Use ChatGPT to help design and then a Cursor prompt to apply changes.
 - Keep the design complete enough that implementation is straightforward.

3. **IPC spec**
 - If the feature crosses process boundaries, update:
   - Pulse IPC specs,
   - Signal IPC specs,
   as needed.
 - Use a dedicated Cursor prompt to keep IPC consistent and minimal.

4. **Implementation**
 - Switch to the relevant repo:
   - Signal for engine/DSP changes,
   - Pulse for model/IPC/undo/redo,
   - Aura for UI/editor changes,
   - Composer for intelligence/metadata.
 - Use Cursor to scaffold modules, functions and types guided by the spec.

5. **Tests**
 - Add unit/integration tests where appropriate
   (see Section 8).

6. **Manual verification**
 - Run the app, exercise the feature.
 - Note any follow-up doc tweaks that emerge.

7. **Finalize**
 - If the architecture changed meaningfully from the original design,
   update the corresponding Chorus doc(s) to match reality.

---

### 6.2 Implementation-First Fix (Small, Local)

Use this for **small, localised fixes** (e.g. typo, obvious bug, tiny refactor).

1. Verify that the change:
 - does not alter behaviour in a way that conflicts with Chorus.
2. Implement directly in the relevant repo using Cursor.
3. Add or adjust tests as needed.
4. If you notice a mismatch with the architecture/spec, record it in:
 - `architecture-backlog.md`, or
 - the appropriate architecture doc as a “Notes / Future cleanup” bullet.

If a “small fix” reveals a broader architectural gap, escalate to
architecture-first.

---

### 6.3 Bug Fixes (Behaviour vs Spec)

For bugs where behaviour clearly contradicts the spec:

1. Confirm desired behaviour in Chorus docs.
2. If the doc is ambiguous:
 - resolve the ambiguity **first** (update Chorus),
 - then fix implementation.
3. Implement the fix in the appropriate repo.
4. Add regression tests.
5. Optionally add a small note in the doc “Known pitfalls / gotchas”
 if the bug exposed a subtle edge case.

---

## 7. Using Cursor Effectively

### 7.1 For Documentation Changes (Chorus)

When using Cursor in Chorus:

- Work in **one file or tightly-related set of files at a time**.
- Prompts should:
- identify exactly which files to edit,
- include clear instructions not to touch `docs/reports/`,
- request a new report file summarising changes.

Pattern:

> - Make these changes to docs X and Y;  
> - Do not modify existing reports;  
> - Create a new report at `docs/reports/<timestamp>-…`.

### 7.2 For Code Changes

In Signal/Pulse/Aura/Composer:

- Start from a **clear, minimal prompt**:
- include relevant architecture snippets (if needed),
- specify which files are in scope,
- specify whether tests should be added/updated.

- After Cursor edits:
- run tests locally,
- compile/run the app,
- manually inspect key changes.

### 7.3 Prompt Reuse

For frequently repeated operations (e.g. adding a new IPC domain, renumbering
docs, updating index), keep reusable prompt templates in:

- `docs/meta/ai-workflow.md`  
- or a dedicated prompts file if useful.

---

## 8. Testing & Validation Workflow

Testing is split across repos, but guided by the architecture.

### 8.1 Pulse (Rust)

- **Unit tests**
- core data structures: tracks, clips, lanes, nodes, routing,
- command handlers and undo/redo,
- versioning and snapshot logic.

- **IPC tests**
- round-trip encode/decode for key Pulse IPC messages,
- compatibility tests for schema evolution.

- **Scenario tests**
- series of edits applied to a project produce expected final model,
- load/save cycles preserve project semantics.

### 8.2 Signal (C++/JUCE)

- **Graph tests**
- node ordering and routing correctness,
- cohort transitions and anticipative rendering scenarios.

- **Render tests**
- offline renders compared to golden reference outputs,
- plugin hosting lifecycle tests.

- **Realtime safety**
- tests (and static analysis where possible) to ensure:
  - no heap allocations on the audio thread,
  - no locks/contention on realtime paths.

### 8.3 Aura (Electron/TS)

- **Unit tests**
- reducers/stores,
- IPC handlers,
- small UI logic components.

- **Integration tests**
- editor flows (arranger, piano roll, clip launcher) using local harnesses.

- **Manual testing**
- always part of the flow for UX-heavy changes.

### 8.4 Composer

- **Determinism & reproducibility (where meaningful)**
- metadata classification tests,
- replacement mapping stability.

- **Privacy checks**
- tests that ensure Composer integration never sends forbidden data fields.

---

## 9. Decision Records & Traceability

All major architectural decisions are recorded as decision docs in:

- `docs/decisions/` (Chorus)

When applying a change that **contradicts or supersedes** an existing decision:

1. Update or add a decision document.
2. Reference it from:
 - relevant architecture docs,
 - relevant backlog entries (if appropriate).

This keeps a **narrative history** of the project’s evolution.

---

## 10. Summary

The implementation workflow for Loophole ensures that:

- **Chorus is the design brain**,  
- **Signal, Pulse, Aura and Composer are the execution limbs**,  
- **ChatGPT** designs and reviews,  
- **Cursor** edits and implements,  
- **You** own direction, integration and final judgement.

By:

- insisting on architecture-first for meaningful changes,
- using clear, consistent prompting patterns,
- enforcing immutability of historical reports,
- and treating testing as an architectural concern,

the project can grow ambitious features without losing coherence or reliability.

This document should be revisited as soon as:

- Pulse’s Rust implementation is underway,
- the first end-to-end IPC flows are live,
- and real-world iteration patterns emerge.

At that point, the workflow can be refined with concrete examples and updated
best practices from actual development experience.
