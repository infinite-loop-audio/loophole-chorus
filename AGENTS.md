# AGENTS

This file defines how AI agents (and future contributors) should work with the
**loophole-chorus** repository.

Chorus is the **meta / architecture / specification** repo for the Loophole
ecosystem. It is the canonical place for:

- cross-repo architecture,
- IPC specifications,
- decision records (ADRs),
- higher-level plans and meta process docs.

The goal is: **highly structured, consistent, precise documentation** that
other tools (and humans) can reliably work with.

---

## 1. General Rules

- Use **British English** spelling.
- Do **not** use emojis in project documentation.
- Write with a clear, technical but approachable tone.
- Prefer **short paragraphs** and clear sectioning over walls of text.
- Avoid trailing whitespace and **do not add a trailing newline** at the end of
  files you create or edit.
- Do **not** create ad-hoc Markdown files in the repo root; use the appropriate
  `docs/` subdirectory.

When in doubt, follow the patterns already present in:

- `docs/architecture/*`
- `docs/specs/*`
- `docs/decisions/*`
- `docs/meta/*`

---

## 2. Linking and Cross-Repo References

### 2.1 Within the same repo (Chorus)

- Use **relative links** for documents within this repository, for example:

  ```markdown
  See [Pulse Architecture](../architecture/a02-pulse.md).
  ```

- Keep relative paths correct if you move or rename files, and update any
  references that point to them.

### 2.2 Between different repos

- For references to other Loophole repos (Signal, Pulse, Aura, Composer), use
  **absolute GitHub URLs**, for example:

  ```markdown
  [Pulse Implementation Plan](https://github.com/infinite-loop-audio/loophole-pulse/blob/main/docs/plans/implementation.md)
  ```

- Do not use relative paths that cross repo boundaries.

### 2.3 URLs vs link text

- Prefer link text over bare URLs:

  ```markdown
  See the [IPC Overview](./ipc/overview.md) for details.
  ```

- Only use bare URLs where there is no meaningful link text.

---

## 3. Architecture Docs (`docs/architecture`)

Chorus holds the **global architecture** for the Loophole ecosystem.

### 3.1 File naming and numbering

- Files live in `docs/architecture/`.
- Use a **two-digit numeric prefix** plus a kebab-cased slug:

  ```text
  01-overview.md
  02-signal.md
  03-pulse.md
  ...
  ```

- The canonical list and ordering is maintained in:

  ```text
  docs/architecture/00-index.md
  ```

- When adding a new architecture doc:
  - Choose the next appropriate number and slug.
  - Add an entry to `00-index.md` with:
    - the filename, and
    - a concise human-readable title.

### 3.2 Document structure

Each architecture doc should:

- Start with a single `#` heading that describes **what** the document is, not
  just the filename, e.g.:

  ```markdown
  # Pulse Architecture
  ```

- Include a **Contents** section for longer documents:

  ```markdown
  ## Contents

  - [1. Overview](#1-overview)
  - [2. Responsibilities](#2-responsibilities)
  ...
  ```

- Use **incremental heading levels**:
  - One `#` at the top.
  - `##` for main sections.
  - `###` / `####` for subsections.
  - Do not skip levels (no `####` immediately after `##`).

- Explain **scope and boundaries** clearly:
  - What this component owns.
  - How it interacts with other components.
  - What is deliberately out of scope.

- Where relevant, cross-link to:
  - related architecture docs (relative links),
  - implementation plans in other repos (absolute URLs).

### 3.3 Renumbering

- Do **not** casually renumber architecture files.
- If renumbering is required:
  - Update `00-index.md` first to reflect the new ordering.
  - Update filenames and all links that reference them.
  - Produce a report in `docs/reports/` describing what changed (see
    [Reports](#6-reports-docsreports)).

---

## 4. IPC Specs (`docs/specs`)

Chorus holds IPC specifications for Pulse, Signal, Aura, Composer, and any
supporting domains.

### 4.1 Structure

IPC specs live under:

```text
docs/specs/ipc/
```

with subfolders by owner or layer, for example:

```text
docs/specs/ipc/pulse/
docs/specs/ipc/signal/
docs/specs/ipc/aura/
```

Each domain spec should:

- Have a top-level `#` heading naming the domain and owner, e.g.:

  ```markdown
  # Pulse Client Domain Specification
  ```

- Start with an **Overview** explaining:
  - the purpose of the domain,
  - what it owns,
  - what it explicitly does not own.

- Include a **Contents** table for non-trivial docs.

- Provide **separate sections** for:
  - Commands (client → server).
  - Events (server → client).
  - Error handling.
  - Notes / constraints.

### 4.2 Commands and events

For each command:

- Specify:
  - envelope `domain` and `type` (or `command` naming pattern),
  - expected payload shape with example JSON,
  - behaviour and side effects,
  - any related events that are emitted.

For each event:

- Specify:
  - envelope `domain` and `type` (or `event` naming pattern),
  - payload shape with example JSON,
  - when it is emitted and what it signifies.

Use realistic examples that are consistent with the rest of the IPC docs.

### 4.3 Ownership

- **Do not** put implementation details from individual repos into IPC specs.
- Specs describe **observable behaviour over IPC**, not internal struct names,
  crate layouts, or type signatures.
- Implementation details (e.g. Rust types, C++ classes) belong in the
  respective repo’s own documentation.

---

## 5. Decisions (`docs/decisions`)

Decision documents (ADRs) record **long-lived architectural choices**.

### 5.1 Template and metadata

- Use `docs/decisions/TEMPLATE.md` as the canonical template.
- Each decision file must include metadata at the top:

  ```markdown
  **ID:** 2025-11-19-pulse-core-execution-model  
  **Date:** 2025-11-19  
  **Status:** proposed | accepted | rejected | superseded  
  **Owner:** <name>  
  **Related docs:**
  - <links...>
  ```

- The filename should mirror the ID:

  ```text
  docs/decisions/2025-11-19-pulse-core-execution-model.md
  ```

### 5.2 Section ordering

Follow the standard section ordering:

1. Context
2. Problem Statement
3. Options
4. Decision
5. Rationale
6. Consequences
7. Follow-Up Actions
8. Notes (optional)

Each section should be a `##` heading.

### 5.3 Content guidelines

- Describe options honestly, with pros and cons.
- Make the **Decision** section explicit and concise.
- Keep references to implementation files via links where useful.
- Do not treat ADRs as scratchpads; they are permanent records.

---

## 6. Meta Docs (`docs/meta`)

Meta documents cover:

- project-wide workflows,
- architecture inbox / backlog,
- repository overviews,
- coordination patterns between tools and agents.

Key files include:

- `docs/meta/ai-workflow.md` — how AI tools should cooperate.
- `docs/meta/architecture-inbox.md` — free-form idea capture.
- `docs/meta/architecture-backlog.md` — structured backlog of future
  architecture work.
- `docs/meta/REPO-OVERVIEW.md` (or similar) — current repo structure summary.

When updating meta docs:

- Preserve their intent (do not turn them into full-blown specs).
- Keep entries short and to the point.
- Do not silently delete existing entries; mark them as done or move them to
  backlog as appropriate.

There may also be a `deprecated/` subfolder:

- Do not remove files from `deprecated/` unless explicitly instructed.
- If you deprecate a process or document, **move** it into `deprecated/` and
  add a short header explaining why.

---

## 7. Reports (`docs/reports`)

Use `docs/reports/` for **one-off analysis or migration reports** produced by
agents (e.g. renumbering reports, refactor summaries, verification reports).

### 7.1 Location and naming

- All new report files must be written under:

  ```text
  docs/reports/
  ```

- Use a **timestamped filename**:

  ```text
  docs/reports/YYYY-MM-DD-HHMMSS-report-name.md
  ```

  Examples:

  ```text
  docs/reports/2025-11-18-163740-architecture-renumber-alignment-report.md
  docs/reports/2025-11-19-120000-architecture-summary.md
  ```

- `report-name` should be short, kebab-case, and descriptive.

### 7.2 Do not touch old reports

- Existing files in `docs/reports/` are **historical artefacts**.
- Do **not** modify or delete reports unless explicitly instructed to do so.
- When you rerun an analysis, create a **new** report with a new timestamp.

---

## 8. Style and Formatting Details

- Only **one `#` heading** per document (the title).
- Ensure heading hierarchy is consistent (`##`, then `###`, etc.).
- Add **Contents** sections to longer documents and keep them reasonably up to
  date.
- Use fenced code blocks with standard triple backticks:

  ```markdown
  ```json
  { "example": true }
  ```
  ```

- Use language tags (`json`, `rust`, `bash`, etc.) where appropriate.

### 8.1 ASCII art and banners

- ASCII art (e.g. for README headers) should **not** be generated by agents.
- If a document needs a banner and one doesn’t exist yet, insert a small
  placeholder and let a human fill it later, for example:

  ```markdown
  <pre>
  CHORUS
  L O O P H O L E
  </pre>
  ```

- Do not attempt to “fix” existing ASCII art unless explicitly asked to.

---

## 9. Behaviour for AI Agents

When acting as an AI agent against this repo:

- Prefer **small, focused changes** over sweeping edits.
- Always update **index files** (`00-index.md`, etc.) when adding/removing
  architecture docs.
- When renaming or moving files:
  - update all references,
  - consider adding a short report in `docs/reports/` describing the change.
- Never introduce `src/`-level code into Chorus; this repo is documentation and
  specification only.
- When unsure about where a new document belongs:
  - Global / cross-repo architecture → `docs/architecture`
  - IPC protocol details → `docs/specs/ipc`
  - Long-lived decision → `docs/decisions`
  - Process / workflow / coordination → `docs/meta`
  - One-off analysis → `docs/reports`

Keep documentation predictable. Future tools (and humans) should be able to
reason about the whole system by reading Chorus without surprises.
