# Versioning & Compatibility Guidelines

This document defines the versioning strategy for specifications and IPC
contracts in Loophole, with a focus on compatibility between **Signal**,
**Pulse**, and **Aura**.

Specifications in `@chorus:/docs/specs/` MUST follow these guidelines.

---

# 1. Scope

Versioning rules apply to:

- JSON schemas (`*.schema.json`)
- TypeScript declaration specs (`*.d.ts`)
- IPC message sets
- Data model formats used for project persistence

---

# 2. Goals

The versioning system MUST:

- Allow independent evolution of Signal, Pulse, and Aura
- Avoid unnecessary breaking changes
- Provide clear upgrade paths
- Support older projects where possible
- Stay simple enough for regular use

---

# 3. Versioning Model

Loophole adopts a **semantic-style versioning** concept for specs:

```
MAJOR.MINOR.PATCH
```

Where:

- **MAJOR** — incompatible changes to schemas or semantics
- **MINOR** — backwards-compatible additions or clarifications
- **PATCH** — editorial or non-structural fixes

Version numbers MAY appear:

- In schema `$id` fields
- In top-level comments/headers in Markdown specs
- In version tables for each spec family

---

# 4. Backwards Compatibility Rules

## 4.1 Additive Changes (Preferred)

The following are considered **backwards-compatible**:

- Adding new optional fields
- Adding new message types that do not alter existing ones
- Adding new enums where old values remain valid
- Relaxing overly strict constraints where safe

## 4.2 Breaking Changes

The following are **breaking changes**:

- Renaming or removing fields
- Changing field types
- Changing required/optional status
- Modifying semantics of existing fields
- Changing message ordering assumptions

Breaking changes MUST:

1. Be documented in an ADR.
2. Trigger a MAJOR version bump for affected specs.
3. Include upgrade guidance.

---

# 5. Deprecation

Fields or message types MAY be deprecated.

Rules:

- Deprecated items MUST be clearly marked in:
  - the schema (where possible)
  - the surrounding Markdown docs
- Deprecation SHOULD precede removal by at least one MAJOR version cycle.
- Deprecation notes MUST include:
  - replacement fields/messages
  - target removal version (if known)

---

# 6. Cross-Repo Compatibility

Each runtime repo (Signal, Pulse, Aura) SHOULD document which spec version(s)
it supports.

For example:

- Signal supports telemetry schema `v1.x`
- Aura supports telemetry schema `v1.x` and `v2.x`

Compatibility matrices MAY be introduced later to track this.

---

# 7. Persistence and Migration

Project file formats managed by Pulse MUST:

- Embed a version identifier
- Provide migration paths between versions
- Treat migration as a first-class operation

Migrations SHOULD be:

- deterministic
- reversible when possible
- logged and traceable

---

# 8. Documentation Requirements

Each spec family SHOULD have:

- A version history section
- A list of breaking changes
- Links to relevant ADRs

Example header:

```markdown
**Current Version:** 1.2.0
**Last Breaking Change:** 1.0.0 (see ADR 0005)
```

---

# 9. Process for Making a Breaking Change

1. Raise an ADR describing:
   - Motivation
   - Impact
   - Alternatives
   - Migration strategy
2. Update affected schemas and docs in `docs/specs/`.
3. Update version metadata.
4. Implement support in Signal, Pulse, Aura.
5. Add tests/validation where applicable.

---

# 10. Summary

The versioning rules in this document ensure the Loophole ecosystem can evolve
without chaos. All specifications must respect these guidelines so that
independent development in Signal, Pulse, and Aura remains safe and predictable.

---
