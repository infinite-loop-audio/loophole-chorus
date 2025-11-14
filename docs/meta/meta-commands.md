# Meta-Commands — Operational Semantics for Meta Blocks

This document defines the **exact behaviour** required for each Meta Block
operation defined in the Meta-Protocol. These operational rules are *normative*
and MUST be followed by all AI agents (Cursor, Codex, ChatGPT middle-agents).

This document complements:

- `meta-protocol.md` — governance & high-level rules
- `meta-schema.json` — formal schema definition
- `.cursor/rules/00-architecture.md` — general editing policy

Together, these form the complete contract for modifying documentation and specs
in the **Chorus** repository.

---

# 1. General Requirements

Before performing any operation, an AI agent MUST:

1. Validate the Meta Block against `meta-schema.json`
2. Confirm the target file exists (unless using `create_file`)
3. Apply the operation **exactly** as defined here
4. Avoid changes outside the scope of the operation
5. Preserve all existing formatting except where required

Agents MUST NOT:

- Auto-correct typos outside targeted content
- Reflow text or normalise whitespace
- Adjust unrelated headings or sections
- Infer content not explicitly provided
- Perform multiple operations from one Meta Block

Each Meta Block represents a **single atomic mutation**.

---

# 2. Operation: `create_file`

### Purpose
Create a new file at the specified `target` path with content exactly matching
the provided `content`.

### Fields
Required:
- `target`
- `content`

### Behaviour
- If the file does **not** exist:
  - Create the file with the content exactly as written.
- If the file **already exists**:
  - Agents MUST NOT overwrite it unless explicitly instructed.
  - The operation MUST fail safely (no changes).

### Formatting Rules
- Do not add, trim, or modify whitespace beyond what appears in `content`.
- Preserve newlines as provided.
- Do not wrap or reflow text.

### Example
```json
{
  "op": "create_file",
  "target": "docs/specs/ipc/pulse/changes.md",
  "content": "# Pulse Change Operations\n\n(Initial content...)"
}
```

---

# 3. Operation: `append_section`

### Purpose
Append a full Markdown section to the end of the specified file.

### Fields
Required:
- `target`
- `content`

### Behaviour
An appended section MUST:

- Begin with a Markdown heading (`#`, `##`, `###`, etc.)
- Be added **after at least one blank line** from the end of the file
- Not modify any existing content

Agents MUST NOT:

- Merge with a previous section
- Auto-indent, reformat, or alter heading levels
- Append without a separating blank line

### Example
```json
{
  "op": "append_section",
  "target": "docs/architecture/01-overview.md",
  "content": "## Plugin Hosting\n\nDescription of plugin hosting..."
}
```

---

# 4. Operation: `replace_section`

### Purpose
Replace a defined Markdown section — identified by its heading — with new content.

### Fields
Required:
- `target`
- `section`
- `content`

### Section Matching Rules
Agents MUST locate the heading line matching `section` **exactly**, including:

- Heading level (`##`, `###`, etc.)
- Spacing
- Punctuation
- Capitalisation

### Replacement Range
The replaced block MUST include:

- The heading line itself
- Every line until:
  - the next heading of the **same or higher** level, OR
  - end of file

### Behaviour
- Replace the entire matched section with `content` exactly
- Do not adjust surrounding whitespace outside the replacement range
- Do not alter neighbouring headings or sections

### Example
```json
{
  "op": "replace_section",
  "target": "docs/architecture/01-overview.md",
  "section": "### Aura (User Interface Layer)",
  "content": "### Aura (User Interface Layer)\n\nUpdated explanation...",
  "notes": "Clarifies interaction boundaries."
}
```

---

# 5. Operation: `delete_section`

### Purpose
Remove a full Markdown section identified by a heading.

### Fields
Required:
- `target`
- `section`

### Behaviour
Deletion operates identically to `replace_section`, except that the matched
section is removed instead of replaced.

Agents MUST:

- Remove the heading and its entire section block
- Leave no additional whitespace beyond one blank line
- Not merge adjacent sections unintentionally

### Example
```json
{
  "op": "delete_section",
  "target": "docs/architecture/legacy.md",
  "section": "## Deprecated Components"
}
```

---

# 6. Operation: `rename_heading`

### Purpose
Rename a single Markdown heading without altering the section content.

### Fields
Required:
- `target`
- `section`
- `content`

### Behaviour
- Locate the heading line matching `section` exactly.
- Replace only that line with `content`.
- Preserve the section’s internal content entirely unchanged.

Agents MUST NOT:

- Change heading level unless instructed
- Modify whitespace following the heading
- Alter any part of the section body

### Example
```json
{
  "op": "rename_heading",
  "target": "docs/specs/ipc-semantics.md",
  "section": "## Timing",
  "content": "## Message Timing Model"
}
```

---

# 7. Operation: `update_schema`

### Purpose
Update a JSON or TypeScript schema.
This operation handles both full replacements and “structured patches”.

### Fields
Required:
- `target`
- `content`

Optional:
- `notes` — human context for reviewers

### Behaviour
Agents MUST:

- Treat schemas as authoritative
- NOT reorder keys unless absolutely necessary
- NOT auto-format JSON beyond what is provided
- NOT introduce trailing commas or comments
- Replace content exactly as the Meta Block specifies

Two modes exist depending on `content`:

## 7.1 Full Replacement (common case)

If `content` contains full schema text:

- Completely replace the file contents
- Do not merge
- Do not infer
- Do not preserve fragments

## 7.2 Fragment Update (with instructions)

If `notes` explicitly instruct merging:

- Insert, remove, or replace only specified schema properties
- Leave all other parts unchanged

### Example — Full Replacement

```json
{
  "op": "update_schema",
  "target": "docs/specs/ipc/signal/telemetry.schema.json",
  "content": "{ \"type\": \"object\", \"properties\": { \"rms\": { \"type\": \"number\" } } }"
}
```

### Example — Structured Patch

```json
{
  "op": "update_schema",
  "target": "docs/specs/ipc/pulse/project.schema.json",
  "content": "{ \"properties\": { \"tempo\": { \"type\": \"number\" } } }",
  "notes": "Merge: update only the tempo property"
}
```

---

# 8. Error Handling

AI agents MUST abort the operation (making no changes) when:

- The target file does not exist (except for `create_file`)
- The `section` heading cannot be found
- The Meta Block does not validate
- Ambiguity exists in selection of content to replace
- JSON parsing fails for `update_schema`
- TS type definitions cannot be resolved

Agents MUST NOT attempt recovery or guessing.

---

# 9. Commit Messages

Agents SHOULD use one of the following prefixes:

- `chore(meta): ...`
- `chore(specs): ...`
- `chore(docs): ...`

Commit messages MUST reference the operation performed:

Example:

```
chore(specs): update telemetry schema (update_schema)
```

---

# 10. Summary

This document provides the exact operational semantics for all Meta Block
operations. AI agents MUST follow these rules precisely to ensure:

- determinism
- correctness
- traceability
- architectural coherence
- consistency across repos

This document, together with `meta-protocol.md` and `meta-schema.json`,
defines the complete mutation model for Chorus.

---
