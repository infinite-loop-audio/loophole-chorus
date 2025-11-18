# AI Workflow Guide  
_A practical guide for working with ChatGPT and Cursor in the Loophole project._

This document defines how **ChatGPT**, **Cursor**, and **you** collaborate during the design and implementation of the Loophole DAW. It replaces the earlier Meta Protocol system, which is now considered **deprecated** for documentation work.

The goal:  
A workflow that is **fast**, **reliable**, **structured**, and **predictable**, without the overhead of rigid schemas or Meta Blocks.

---

## 1. Principles

The Loophole documentation and code ecosystem is built on three pillars:

### **1.1 ChatGPT = Architect & Systems Designer**
ChatGPT is responsible for:
- designing architecture and specifications,
- reasoning about trade-offs,
- proposing protocols and workflows,
- maintaining conceptual consistency across the entire project.

ChatGPT does **not** directly manipulate files in the repo — that is Cursor’s job.

### **1.2 Cursor = Precise Editor & Refactor Tool**
Cursor is used to:
- create, modify, or move files,
- apply consistent formatting rules,
- run consolidation or cohesion passes,
- generate reports on file changes.

Cursor performs **all** edits to the repo based on instructions we give it.

### **1.3 The Developer (you) = Reviewer & Orchestrator**
You:
- request architectural output from ChatGPT,
- give Cursor high-level instructions,
- review changes,
- push commits,
- maintain momentum and prioritise tasks.

This triad is extremely efficient when used correctly.

---

## 2. Directory Conventions and Formatting Rules

These rules are global and must always be followed.

### **2.1 British English**
All content, including documentation and comments, uses **British English** spelling.

### **2.2 Quad Backticks for File Output**
ChatGPT must wrap **full-file outputs** in quadruple backticks:

````markdown
# file.md
content...
````
Inner triple-backtick code blocks are allowed.

### **2.3 No trailing newline**
All files end immediately after the last line, with **no trailing newline**.

### **2.4 Absolute cross-repo links**
Within GitHub-facing documentation:
- Links to files **in the same repo** → _relative links_  
- Links to files **in other Loophole repos** → _absolute URLs_

### **2.5 Document Naming**
- Architecture docs: `docs/architecture/NN-title.md`  
- IPC docs: `docs/specs/ipc/<pulse|signal>/<domain>.md`  
- Decision records: `docs/decisions/YYYYMMDD-title.md`  
- Meta docs: `docs/meta/<file>.md`

Cursor maintains naming when modifying or adding files.

---

## 3. Typical Collaboration Flow

### **3.1 For new architecture/specification work**
1. You: ask ChatGPT for a new domain design or expansion.  
2. ChatGPT: produces the full document in quad ticks.  
3. You: ask Cursor to create/update the file.  
4. Cursor: writes the file and reports changes.  
5. You: commit.

### **3.2 For updating existing documents**
1. You: ask ChatGPT for the *desired change*, not the diff.  
2. ChatGPT: produces a **Cursor task prompt**, usually phrased as:  
   “Update `<file>` by applying changes X, Y, Z…”
3. You: paste that into Cursor.  
4. Cursor: updates files & reports.  
5. You: review → commit.

### **3.3 For large multi-file refactors**
Use the pattern:

- “Perform a cohesion pass”
- “Perform a consistency pass”
- “Perform a reorganisation pass”
- “Convert all remaining references of X to Y”

Always request a final report file to review output.

---

## 4. Inbox and Backlog

### **4.1 Architecture Inbox**
Used for **raw ideas**.

- Cursor appends informal entries automatically.
- ChatGPT does not touch this directly.
- You periodically review and promote ideas into backlog items or new specs.

### **4.2 Architecture Backlog**
Used for **structured future work**.

- Items remain until implemented.
- You and ChatGPT pull from it when deciding next tasks.

### **4.3 When to add to inbox vs backlog**
- **Inbox**: fuzzy ideas, sketches, concepts, half-formed features.  
- **Backlog**: validated features, architectural work, or required follow-ups.

---

## 5. Cursor Task Patterns

### **5.1 Cohesion Pass**
Used when:
- many files need consistency,
- naming or structural drift has occurred.

Typical prompt:

> Perform a cohesion pass on all IPC docs.  
> Ensure envelope semantics, naming, and field shapes match.

### **5.2 Structure Pass**
Used when reorganising or renaming files.

### **5.3 Append Mode**
Used for inbox entry adding.

### **5.4 Direct Editing Mode**
Used for isolated changes:
- “Update this file to fix section X”
- “Insert the following paragraph after Y”
- “Replace all references to A with B”

### **5.5 Report Files**
For all multi-file edits, Cursor must produce a report in:

```
docs/meta/<TASK>-REPORT.md
```

This ensures full auditing.

### **5.6 Report File Storage**

All multi-file Cursor operations (cohesion passes, renumbering, refactors, link rewrites, protocol updates, etc.) must produce a report file stored in:

```
docs/reports/
```

Report files must follow the naming convention:

```
YYYY-MM-DD-HHMMSS-task-name-report.md
```

Where `YYYY-MM-DD-HHMMSS` is the UTC timestamp at the moment the task is run. This format is chosen so that lexicographic filename order matches chronological order for reports created on the same day.

Rules:

- All reports go into `docs/reports/`.
- One report per major Cursor task.
- No trailing newline.
- Cursor must create `docs/reports/` if it does not exist.

---

## 6. Decision Documents (ADRs)

Whenever a substantial architecture or system-level decision is made:

- ChatGPT produces a new decision document in the template format.
- Cursor creates the file.
- The decision is final unless superseded by a future ADR.

Use ADRs when:
- choosing a language,
- defining core engine behaviour,
- finalising IPC design,
- solidifying architectural patterns.

---

## 7. ChatGPT Output Rules

ChatGPT must:

- always use British English spelling,
- never introduce emojis,
- produce whole files in quadruple backticks,
- label files clearly (filename at the top),
- never mirror or rewrite full repo contents,
- keep outputs concise unless detailed content is required,
- always ask before making destructive or large-scale changes.

---

## 8. Cursor Output Requirements

Cursor must:

- only modify the files requested,
- preserve formatting style,
- produce a file change report,
- never insert trailing newlines,
- update internal links after file renames.

Cursor is **never** responsible for designing architecture — only for applying edits.

---

## 9. Deprecated: Original Meta Protocol

The older files:

- `meta-protocol.md`
- `meta-commands.md`
- `meta-schema.json`

These are preserved for historical reference but no longer govern the workflow.

They may be revived later for:
- logic-level schema transformations,
- batch spec refactors during implementation,
- code-level AST transformations,
- safe mutation of formal schemas.

For documentation and design phases, they are considered **superseded** by this `ai-workflow.md`.

---

## 10. Summary

The workflow is intentionally simple:

**ChatGPT designs.  
Cursor edits.  
You review.**

This document codifies that process so it remains consistent for the entire lifespan of the Loophole project.
