# Specifications Overview

This directory (`@chorus:/docs/specs/`) contains **all authoritative,
machine-oriented specifications** for the Loophole ecosystem.
These specifications define the **contracts** between runtime components —
Signal, Pulse, and Aura — and must be treated as *binding* by all repos.

The goal of this folder is to:

- Maintain strict, predictable communication formats
- Provide machine-readable schemas for AI-assisted tooling
- Ensure compatibility between independently evolving processes
- Prevent undocumented assumptions across the system
- Serve as the single source of truth for IPC and data structures

This folder is the **contractual core** of Loophole.

---

# 1. Purpose of Specifications

Specifications in this directory govern:

- **Inter-process communication (IPC)**
  Formats used between:
  - Aura → Signal
  - Signal → Aura
  - Aura → Pulse
  - (future) Pulse → Signal
  - (future) Pulse → remote workers

- **Data structures**
  Common project-level JSON/TS structures for:
  - Plugin descriptors
  - Track definitions
  - Routing structures
  - Automation curves
  - Parameter models
  - Telemetry packet formats

- **Schema guarantees**
  - Validation rules
  - Required and optional fields
  - Type constraints
  - Backward compatibility expectations

---

# 2. Specification Types

The following document formats are supported and expected:

## 2.1 JSON Schemas

Used for highly formalised, programmatically validated contracts:

- IPC messages
- Telemetry frames
- Graph update instructions
- Plugin metadata formats

Always named using:

```
*.schema.json
```

JSON Schemas MUST:

- Use Draft-07 or later
- Use concise, stable property ordering
- Never include comments (JSON does not support them)
- Provide `$id` where applicable
- Reference sub-definitions via `$ref` for reuse

---

## 2.2 TypeScript Interfaces

Used when human readability and tooling integration take precedence (particularly for Pulse & Aura):

```
*.d.ts
```

TS interface spec files must:

- Use only type-level declarations
- Not include runtime code
- Alphabetise keys in interfaces
- Prefer `readonly` properties
- Avoid TypeScript-specific complexity unless required (e.g., generics)

---

## 2.3 Markdown Specifications

Used for:

- Topic guides
- IPC semantics
- Operational expectations
- Notes requiring prose + examples

Markdown spec files should:

- Use stable section headings
- Include fenced triple-backtick code blocks for examples
- Reference JSON/TS schemas via relative paths
- Distinguish normative language:
  - MUST / MUST NOT
  - SHOULD / SHOULD NOT
  - MAY

---

## 2.4 C/C++ Struct Definitions

Used sparingly for Signal’s IPC (e.g., binary telemetry streams).

These MUST appear only inside fenced code blocks:

```c++
struct MeterFrame {
    float peak;
    float rms;
    float crest;
};
```

They MUST NOT introduce implementation detail beyond what is used for IPC.

---

# 3. Directory Structure

The following structure is recommended (and expected) as the directory grows:

```
@chorus:/docs/specs/
  ipc/
    signal/
      commands.schema.json
      telemetry.schema.json
      handshake.schema.json
    pulse/
      project.snapshot.schema.json
      change-operations.schema.json
  data/
    plugin-descriptor.schema.json
    track.schema.json
    routing.schema.json
    automation.schema.json
  guidelines/
    ipc-semantics.md
    versioning.md
    realtime-safety.md
```

You may initialise subfolders now, or allow them to grow as specs appear.

---

# 4. Specification Ownership and Change Control

## 4.1 Specifications are binding

Every repository in the Loophole ecosystem MUST conform to the schemas in this directory.
Changes to this folder must be approached with extreme care.

## 4.2 Changes MUST follow one of the following paths:

### **Path A (preferred)** — Meta-Protocol Update

- A Meta Block is generated
- Cursor/Codex applies it
- Human reviews the diff
- Commit follows `chore(specs): ...` format

### **Path B** — ADR-Driven Change

For breaking or structural changes:

1. Write a new ADR in `@chorus:/docs/decisions/`
2. Update specs accordingly
3. Update related architecture docs
4. Cross-link where appropriate

### **Path C** — Manual Update (rare)

Only used for small clarifications.
Still requires human review.

---

# 5. Versioning & Compatibility

Specifications define compatibility boundaries across repos.

Specs MUST follow:

- **Backward compatibility by default**
- Additive changes preferred over destructive ones
- Breaking changes MUST:
  - Include a new ADR
  - Update the versioning guide
  - Indicate compatibility notes inside the schema

The file:

```
@chorus:/docs/specs/guidelines/versioning.md
```

(Generated later) will describe this in detail.

---

# 6. Validation Expectations

All JSON Schema documents MUST be validated using standard tools.
All TS declarations MUST compile under `tsc --strict`.

Signal, Aura, and Pulse repos SHOULD include automated validation pipelines
(in CI) referencing these schemas.

---

# 7. Documentation Requirements

Every specification must include:

- A clear purpose
- A normative rules section
- Examples where relevant
- Cross-links to related specs
- Notes for AI agents (if needed)

---

# 8. Summary

The `@chorus:/docs/specs/` folder defines the **strict contractual backbone** of
the Loophole system. Runtime code must conform to these definitions exactly.
Documentation, architecture, and ADRs must be kept consistent with these specs.

Chorus is the source of truth.
These specifications are the *mechanics* of that truth.

---
