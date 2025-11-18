# Meta-Protocol for Documentation & Specification Updates

> **Status:** Deprecated  
> This file is preserved for historical reference.  
> It has been superseded by `../ai-workflow.md` and is no longer part of the active project workflow.

This document defines the **Meta-Protocol**, a structured mechanism for safely,
predictably, and reproducibly updating documentation and specifications within
the **Chorus** meta-repository.

The Meta-Protocol exists primarily to support **AI-assisted development** using
tools such as ChatGPT, Cursor, and Codex.
It ensures that changes across the Loophole architecture remain:

- deterministic
- explicit
- validated
- reviewable
- traceable

The Meta-Protocol is **normative** (historical; see deprecation header):
All AI agents MUST follow it when modifying documentation or specifications.

---

# 1. Purpose

The Meta-Protocol governs:

- How architectural changes are represented
- How documentation files are updated
- How specification files are modified
- How multi-agent workflows remain consistent
- How AI tools safely apply changes without ambiguity

Its goals include:

- Preventing unintended document drift
- Standardising update formats
- Protecting specification integrity
- Enabling large-scale changes to be applied incrementally
- Ensuring human reviewers understand the intent behind every change

---

# 2. Meta Blocks

A *Meta Block* is a JSON object conforming to the schema:

```
@chorus:/docs/meta/deprecated/meta-schema.json
```

Meta Blocks:

- Describe a **single atomic change**
- Are the only AI-safe mechanism for modifying specs
- MUST be applied exactly as written
- MUST NOT be interpreted or “helped along” by an AI agent

Every Meta Block contains at least:

- `op` — the operation
- `target` — the file path

Certain operations require extra fields (e.g. `section`, `content`).

---

# 3. Supported Operations

Operations define how documentation/specs should be mutated.

These are formally defined in:

```
@chorus:/docs/meta/deprecated/meta-commands.md
```

The approved operations are:

| Operation          | Purpose |
|--------------------|---------|
| `create_file`      | Create a new file with specific content |
| `append_section`   | Append a Markdown section to a file |
| `replace_section`  | Replace a named Markdown section |
| `delete_section`   | Delete a named Markdown section |
| `rename_heading`   | Rename a Markdown heading |
| `update_schema`    | Replace or merge schema fragments |

These six operations form the entire mutation vocabulary for AI agents.

If an operation cannot be expressed using these, the user must revise the Meta Block or add a new operation (via ADR).

---

# 4. Meta Block Examples

## 4.1 Replace a Markdown section

```json
{
  "op": "replace_section",
  "target": "docs/architecture/01-overview.md",
  "section": "## Layer Responsibilities",
  "content": "## Layer Responsibilities\n\n(Updated section...)",
  "notes": "Clarifies boundary between Pulse and Signal."
}
```

## 4.2 Create a new Markdown file

```json
{
  "op": "create_file",
  "target": "docs/specs/ipc/telemetry.md",
  "content": "# Telemetry Specification\n\n(Initial content...)"
}
```

## 4.3 Update a JSON schema file

```json
{
  "op": "update_schema",
  "target": "docs/specs/ipc/signal/telemetry.schema.json",
  "content": "{ \"properties\": { \"fft\": { \"type\": \"array\" } } }",
  "notes": "Adds FFT field to telemetry packet."
}
```

Meta Blocks MUST NOT contain comments inside JSON fields.

---

# 5. Validation Requirements

Before acting on any Meta Block, an AI agent MUST:

1. Validate the block against `../deprecated/meta-schema.json`
2. Reject any block that does not conform
3. Fail safely by making **no changes**

Validation rules include:

- Required fields must exist
- No undeclared fields may be present
- `op` must match the allowed enum
- Data types must match the schema

If validation fails, the human user must supply a corrected Meta Block.

---

# 6. Application Rules for Middle Agents

AI tools (Cursor, Codex, ChatGPT agents) MUST apply Meta Blocks according to:

## 6.1 Single-responsibility

Each Meta Block MUST correspond to exactly one conceptual change.

## 6.2 Deterministic behaviour

The operation MUST produce the same output regardless of:

- agent version
- platform
- time
- environment

## 6.3 No speculative behaviour

Agents MUST NOT:

- infer missing content
- auto-reflow text
- “fix” formatting
- reorganise headings
- change unrelated sections
- update cross-links unless explicitly requested

## 6.4 No opportunistic edits

Agents MUST NOT:

- correct typos outside `target` section
- modify adjacent whitespace unless required
- normalise Markdown unless instructed
- add newlines except as required by the operation

## 6.5 File integrity

Agents MUST preserve:

- newline stability
- indentation
- code block integrity
- paragraph structure
- heading hierarchy

---

# 7. Human Review

Even when operations are executed by AI, all changes to Chorus MUST undergo human review before merging.

Reviewers SHOULD:

- Confirm operation correctness
- Confirm content correctness
- Confirm that scope was correctly limited
- Confirm no hidden changes occurred
- Confirm commit messages follow conventions

A reviewer may request:

- A revised Meta Block
- A new ADR
- Additional architectural context
- A different file structure

---

# 8. Enforcement Model

The Meta-Protocol is binding.

Violations result in:

- rejected PRs
- rollback of changes
- revision of AI agent instructions

All repos in the Loophole ecosystem MUST treat Chorus specifications as authoritative.

---

# 9. When NOT to Use the Meta-Protocol

These cases are exceptions:

- Fixing a **spelling error** (when instructed explicitly)
- Fixing a **broken link**
- Adding a **new ADR** (as these are append-only)

However, *any* change that modifies normative content MUST use a Meta Block.

---

# 10. Extending the Meta-Protocol

New operations MAY be added, but require:

1. A dedicated ADR
2. Updates to:
   - `../deprecated/meta-schema.json`
   - `../deprecated/meta-commands.md`
   - This document
3. A justification for why existing operations are insufficient

The Meta-Protocol must remain minimal and coherent.

---

# 11. Summary

The Meta-Protocol ensures that Chorus:

- remains stable
- evolves intentionally
- supports multi-agent collaboration
- guards against accidental breakage
- keeps the Loophole architecture consistent

All changes to documentation and specs MUST follow this protocol unless explicitly exempted.

Chorus is the root of truth.
The Meta-Protocol is the guardian of that truth.

---
