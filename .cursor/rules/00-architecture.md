# Cursor Rules — Chorus Meta-Repository

These rules instruct Cursor (and other AI middle-agents) on how to safely edit
the **Chorus** repository. Chorus contains *meta-level documentation only* and
defines the architectural contracts for the Loophole ecosystem.

The purpose of these rules is to:

- Maintain cross-repo architectural coherence
- Preserve correctness of specifications
- Ensure Meta-Protocol compliance
- Prevent speculative or unintended edits
- Support deterministic, reproducible AI-assisted development
- Keep documentation consistent, stable, and human-readable

---

# 1. Repository Scope and Boundaries

- This repository MUST NOT contain:
  - Application/runtime code
  - Build scripts, binaries, or libraries
  - Implementations for Signal, Pulse, or Aura
- This repository MUST contain:
  - Architecture overview documents
  - IPC schemas and message definitions
  - Real-time safety guidelines
  - ADRs (Architecture Decision Records)
  - Meta-Protocol definition
  - Task briefs for AI agents
  - Documentation rules and conventions
- Under no circumstances should Cursor introduce C++, TypeScript, or JS code
  belonging to runtime layers.

---

# 2. Editing Philosophy

AI tools should follow these principles:

- **Minimal surface area for each change**
  Only modify what the user explicitly instructs.

- **No speculative modifications**
  Do not “fix”, reorganise, or optimise content unless asked.

- **Human-first clarity**
  Documentation should remain readable by humans first, tooling second.

- **Reproducibility**
  All changes must be deterministic and traceable.

- **Source of truth**
  Specs in `docs/specs/` override any inline description elsewhere.

---

# 3. Editing Architecture Docs (`docs/architecture/`)

These documents describe the conceptual, high-level system structure.

- Keep system boundary definitions consistent:
  - **Signal** → real-time audio engine
  - **Pulse** → project/model state
  - **Aura** → UI and interaction
  - **Chorus** → coordination, contracts, specs
- When describing interactions:
  - Prioritise precision
  - Avoid implementation-specific detail
  - Keep diagrams ASCII-friendly
- Maintain stable section headings.
- If major structural changes are introduced:
  - Add or update an ADR
  - Ensure all other architecture docs remain coherent

---

# 4. Editing Specifications (`docs/specs/`)

These are **contracts** between runtime repos.

AI must:

- Treat these files as authoritative
- Maintain formatting conventions:
  - JSON schema → compact, stable ordering
  - TS interfaces → alphabetised keys
  - Markdown → named sections with explicit fences
- Avoid renaming fields unless explicitly instructed
- Inserting breaking changes requires:
  - A new ADR OR
  - A user-provided Meta Block
- Preserve backward compatibility notes when present
- Update cross-references (e.g. link to ADR or architecture doc) only when instructed

---

# 5. Editing ADRs (`docs/decisions/`)

- ADRs are append-only.
- Existing ADRs must NOT be altered except to:
  - Fix typos (only when instructed)
  - Update status fields (e.g. Accepted → Superseded)
- New ADRs should follow the established template:
  - Status
  - Context
  - Decision
  - Consequences
  - Notes
- ADR filenames follow:
  ```
  ####-short-title.md
  ```

---

# 6. Editing Meta-Protocol Files (`docs/meta/`)

These govern how AI tools update documentation.

AI must:

- Treat `meta-schema.json` as immutable unless explicitly instructed
- Ensure the operations in `meta-commands.md` remain aligned with the schema
- Maintain consistency between:
  - `meta-protocol.md`
  - `meta-schema.json`
  - `meta-commands.md`

When updating these files:

- Explicitly cross-reference related sections
- Avoid informal language
- Avoid unnecessary examples or verbosity
- Use precise definitions and verbs

---

# 7. Editing Task Files (`tasks/`)

Task files represent atomic units of work.

AI must:

- Treat each task as **standalone**
- Not combine or merge tasks unless instructed
- Keep tasks short, actionable, and machine-parsable
- Use headings:
  - Context
  - Meta Block
  - Instructions for Middle Agents

---

# 8. Formatting and Markdown Conventions

- Use **British English** for all documents.
- Use Markdown with:
  - `#` style headers
  - fenced code blocks inside quadruple-backtick containers for file outputs
- Do NOT introduce raw HTML unless absolutely necessary
- Keep lines ≤ 120 characters
- Use consistent heading levels across files

---

# 9. Commit Messages (for agents)

Commit messages applied by Cursor should follow the pattern:

```
chore(docs): <short description>
```

For example:

```
chore(docs): update Pulse process description
chore(specs): refine telemetry packet schema
chore(meta): add rename_heading operation
```

---

# 10. Safety Expectations for AI Tools

Cursor must avoid:

- Overwriting large files unintentionally
- Reformatting entire documents unless instructed
- Altering semantics of specification keys
- Generating runtime code
- Moving or renaming core directories
- “Cleaning up” or restructuring documents unprompted
- Merging multiple conceptual changes into one commit

When in doubt: **prefer no change**.

---

# 11. Enforcement Model

Chorus is the *knowledge system* underpinning all other repos.
Therefore:

- Changes to Chorus must be conservative
- Human review is mandatory
- Meta Blocks are the preferred mechanism for updates
- Cursor must respect the schema and rules exactly

---

# 12. Summary

Chorus must remain:

- stable
- predictable
- precise
- unambiguous
- easy for humans to read
- easy for AI tools to act on

Cursor must treat the contents of this repository as canonical truth for the entire Loophole architecture.

---
