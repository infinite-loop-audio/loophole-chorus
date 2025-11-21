# ADR-XX — Unified IPC Naming Model (Command/Event Convergence)

**Status:** Proposed  
**Date:** 2025-11-21  
**Context:** IPC naming consistency across Chorus, Pulse, and Aura

---

## 1. Problem

The existing IPC naming model mixes:

- command names (`project.open`)
- past-tense event names (`project.opened`)
- “stateChanged” / “graphReset” naming
- `kind=response` envelopes that duplicate event semantics
- inconsistent double-barrel naming rules

This results in:

- semantic inconsistency  
- duplicated naming for command → event  
- brittle client logic (Aura must handle “responses” separately)  
- excessive verbosity  
- harder long-term maintenance  
- inconsistent behaviour between Chorus (spec), Pulse (impl), Aura (impl)

A structural simplification is needed.

---

## 2. Decision

We adopt a **unified naming model** with the following principles.

### 2.1. Domains stay; names simplify

- Each envelope uses:
  - `domain`: subsystem namespace (e.g. `"project"`, `"track"`)
  - `name`: the *operation* in **camelCase**, no tense variants

**Names represent the action itself**, not past-tense or descriptions.

Examples:

- `create` instead of `created`
- `open` instead of `opened`
- `resetGraph` instead of `graphReset`
- `state` instead of `stateChanged`

### 2.2. Commands and events share the *same* name

For a given operation:

```
Command →  domain="track", name="create"
Event   →  domain="track", name="create"
```

They are distinguished by:

- `kind="command"`
- `kind="event"`

### 2.3. Remove `kind=response` entirely

All server replies become **events**.

Correlation is preserved via:

- `cid = <command.id>` when responding directly
- `cid = null` for unsolicited events

This eliminates:

- redundant response naming
- duplicated command/response mapping code
- confusion between event vs response semantics

### 2.4. Minimise double-barrel names

Names should be single words unless required for clarity.

Examples:

- Good:
  - `track.create`
  - `project.open`
  - `transport.seek`

- Necessary:
  - `channel.resetGraph` (because the domain is channel)
  - `engine.flushState` (because “flush” alone is ambiguous)

### 2.5. Standardised state reporting

For all stateful domains:

- Use `name="state"` for incremental state events
- Use `kind="snapshot"` for full-state snapshots

Remove all *“stateChanged”*-style names.

### 2.6. Summary of removed patterns

| Old pattern     | New naming             |
|-----------------|------------------------|
| created         | create                 |
| updated         | update                 |
| opened          | open                   |
| graphReset      | resetGraph             |
| stateChanged    | state                  |
| `kind=response` | removed (use event+cid)|

---

## 3. Rationale

This model:

- dramatically simplifies IPC naming  
- makes commands ↔ events visually paired  
- removes special-case response handling  
- fits naturally with the envelope’s existing `cid` correlation field  
- produces cleaner logs and easier-to-read traffic  
- eliminates tense inconsistencies (open/opened)  
- avoids “double-barrel everywhere” naming fatigue  
- prepares Chorus for large-scale domain growth

Most importantly:  
**it makes the IPC protocol predictable and teachable.**

---

## 4. Consequences

### 4.1. Good

- Specification becomes dramatically simpler  
- Pulse and Aura code shrink (no more duplicated “opened/created”)  
- UI code becomes easier to reason about  
- No more “response” branch logic  
- Natural support for multiple non-command-triggered events  
- Greater consistency across all domains

### 4.2. Neutral / manageable

- Requires renaming large portions of Chorus specs  
- Requires updates across Pulse & Aura implementations  
- Requires tailored migration prompts

### 4.3. Bad / risks

- Downstream clients expecting legacy names will break  
- Debug logs temporarily mixed until migration complete

---

## 5. Alternatives Considered

### A. Keep past-tense events
Rejected — inconsistent, overly verbose, more code required.

### B. Keep `kind=response`
Rejected — redundant with `cid`, complicates dispatcher and clients.

### C. Combine domain & name into a single dotted string
Rejected — type-safety loss, more parsing, more brittle.

---

## 6. Adoption Plan

1. Apply this ADR to Chorus documentation.  
2. Generate migration prompts for:
   - Chorus docs
   - Pulse implementation
   - Aura implementation
3. Enforce via automated tests:
   - every command must have a same-named event
   - no `response` kind allowed
   - names must be camelCase, single-word where possible

---
