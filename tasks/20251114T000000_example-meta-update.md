# Task: Example Meta-Update (Demonstration of Meta-Protocol Use)

This file demonstrates how to structure a task for AI middle-agents using the
Meta-Protocol defined in `@chorus:/docs/meta/`.

Tasks in `@chorus:/tasks/` follow strict conventions:

- Filename begins with a sortable timestamp:
  `YYYYMMDDThhmmss_description.md`
- Document MUST contain:
  - Context
  - Meta Block
  - Instructions for Middle Agents
- Task files are **immutable** once completed (append-only system)

This task is not intended to be executed.
It serves as a template and reference for future real tasks.

---

# 1. Context

We want to demonstrate the usage of a Meta Block by modifying a specific section
inside an architecture document.

This example uses:

- Operation: `replace_section`
- Target: `@chorus:/docs/architecture/01-overview.md`
- Section: `"### Pulse (Data Model)"`
- Purpose: Show the expected formatting and workflow

This task is illustrative only.

---

# 2. Meta Block

```json
{
  "op": "replace_section",
  "target": "docs/architecture/01-overview.md",
  "section": "### Pulse (Data Model)",
  "content": "### Pulse (Data Model)\n\nUpdated description text...",
  "notes": "Demonstration of the Meta-Protocol. Do not execute."
}
```

---

# 3. Instructions for Middle Agents

AI tools (Cursor, Codex, ChatGPT) MUST follow these steps:

1. Validate the Meta Block against
   `@chorus:/docs/meta/meta-schema.json`.

2. DO NOT apply this Meta Block automatically —
   this file is a **demonstration task**, not an actionable change.

3. When processing real tasks:
   - Apply the operation exactly as defined
   - Make no changes outside the target section
   - Preserve formatting, headings, and whitespace
   - Commit with a message:
     ```
     chore(docs): <short summary> (via meta-block)
     ```

4. Do not modify this file except to append a completion note (if desired).

---

# 4. Notes for Human Reviewers

- Each future task should follow this structure exactly.
- Complex changes SHOULD be decomposed into multiple small tasks.
- Completed tasks MAY be marked at the bottom with:
  ```
  **Status:** Completed on YYYY-MM-DD by <name>
  ```

This file acts as the baseline example for all future tasks.

---
