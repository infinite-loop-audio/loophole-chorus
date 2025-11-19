# Decision: Aura UI Framework and Styling Stack

## Metadata

- **ID:** 2025-11-19-aura-ui-framework
- **Date:** 2025-11-19
- **Status:** accepted
- **Owner:** Loophole Architecture
- **Related Docs:**
  - `docs/architecture/a00-overview.md` (Loophole Architecture Overview)
  - `docs/architecture/b02-aura-architecture.md` (Aura Architecture)
  - `docs/plans/aura/implementation.md` (Aura Implementation Plan, planned)

---

## Context

Aura is the desktop UI layer for Loophole: an Electron-based application that
hosts the full DAW interface (timeline, mixer, browsers, editors, clip
launcher, plugin management, diagnostics, etc.).

Requirements for Aura include:

- High performance and responsiveness, especially in:
  - clip and piano-roll editors,
  - mixer and metering views,
  - clip launcher and arrangement views.
- A **clean, easily understandable codebase**, where:
  - each component’s purpose is obvious,
  - files are approachable to new contributors,
  - there are no deep callback/closure forests.
- Strong TypeScript support.
- Clear separation between:
  - project/engine model (Pulse & Signal),
  - UI view state (Aura).
- A theming system suitable for:
  - multiple visual themes (dark/light, future skins),
  - DAW-specific visual vocabulary (tracks, lanes, nodes, etc.).
- A good fit with AI-assisted development (Cursor, ChatGPT), without producing
  over-complicated patterns.

Historically, Vue (with the Options API) has been used in other projects, but
those choices were made years ago under different constraints, and Aura does not
have to inherit them.

We need a deliberate choice of:

1. UI framework (React, Vue, Svelte, Solid, or other),
2. Styling and theming approach (Tailwind vs hand-rolled design system).

---

## Problem Statement

We must choose a UI framework and styling approach for Aura that:

- scales to a large and complex desktop DAW UI,
- produces **readable and maintainable** code over the long term,
- avoids the common failure modes of large TypeScript frontends
  (hook/closure/indirection hell),
- supports strong theming and a coherent design system,
- works well in Electron,
- and plays nicely with AI-assisted coding tools.

The decision is foundational: changing UI frameworks later would be extremely
disruptive.

---

## Options

### Option A — React + Tailwind

**Description**

Use React (hooks-based) for the renderer UI, with Tailwind CSS for styling.

**Pros**

- Huge ecosystem, many Electron + React examples.
- Best-supported target for AI coding tools.
- Tailwind speeds up early prototyping and enforces consistent spacing/typography.
- Many off-the-shelf components and patterns.

**Cons**

- Hooks and Tailwind together often produce:
  - deeply nested closures,
  - indirection-heavy component hierarchies,
  - “class name soup” in templates.
- Readability suffers in large codebases:
  - difficult for newcomers to understand complex components,
  - styling and behaviour are tightly interwoven in markup.
- Theming with Tailwind and arbitrary DAW skins requires:
  - heavy reliance on CSS variables anyway,
  - custom utility layers,
  - discipline that negates much of Tailwind’s advertised simplicity.
- Encourages the very patterns we want to avoid: clever abstractions,
  poorly named hooks, and fragmented state.

**Conclusion**

Rejected. While React + Tailwind is powerful, it conflicts with the goal of a
clean, easily understandable UI layer and makes deep theming unnecessarily
awkward.

---

### Option B — React + Hand-Rolled Design System

**Description**

Use React for the UI, but with a custom design system and CSS variables
instead of Tailwind. Styling would rely on CSS modules or styled-components
and a central theme system.

**Pros**

- Retains React’s ecosystem and AI friendliness.
- Provides clearer theming via design tokens and CSS variables.
- Can be structured to avoid Tailwind’s class clutter.

**Cons**

- React’s hooks model still tends to produce:
  - effect/useMemo/useCallback pyramids,
  - hard-to-follow closure chains in complex interactive components.
- Requires strong, ongoing discipline to avoid common readability pitfalls.
- Component code is less declarative and more imperative/JS-heavy than needed
  for a DAW UI.
- Still relatively heavy at runtime compared to compiled approaches.

**Conclusion**

Rejected. Better than React + Tailwind, but still misaligned with the priority
of maximising clarity and minimising indirection in a complex DAW UI.

---

### Option C — Vue 3 (Options API) + Hand-Rolled Design System

**Description**

Use Vue 3 with the Options API as the dominant component style (Composition API
reserved for shared logic). Styling via CSS variables + scoped styles in
Single-File Components (SFCs), plus a small design system.

**Pros**

- Options API is **very readable** and easy for humans to scan.
- SFCs provide a clean structure: template, script, style in one place.
- Theming via CSS variables is straightforward.
- Scoped component styles avoid cross-component leakage.
- Keeps simple things simple, while still allowing composition where needed.

**Cons**

- Slightly smaller ecosystem for complex editor patterns than React,
  but still adequate.
- AI tools are capable with Vue but not as optimised around it as React.
- Slightly more framework-specific concepts (reactivity caveats, etc.).

**Conclusion**

Strong candidate. Very good readability and structure, but not necessarily the
simplest or lightest runtime for the most demanding, interactive editor views.
Still an acceptable choice, but not the optimal one given alternatives.

---

### Option D — Svelte + Hand-Rolled Design System

**Description**

Use Svelte as the UI framework. Styling via CSS variables, local scoped styles,
and a small shared design system for DAW components.

**Pros**

- Extremely clean component syntax:
  - HTML-like templates,
  - minimal boilerplate,
  - behaviour is obvious at a glance.
- No hooks or complex lifecycle gymnastics:
  - state and behaviour are explicit,
  - minimal risk of callback/closure forests.
- Svelte compiles away framework overhead:
  - very fast rendering,
  - excellent fit for interactive editors and mixers.
- Scoped styles and CSS variables work naturally for theming.
- Encourages a “one file, one concept” approach that aligns with Aura’s goals.
- Easy to reason about even for developers new to the project.

**Cons**

- Smaller ecosystem than React/Vue, but still sufficient for our use case.
- AI tools are a bit less optimised for Svelte than React, but still capable.
- Some contributors may be less familiar with Svelte than with React/Vue.

**Conclusion**

Accepted. Svelte best satisfies the requirements of clarity, maintainability
and performance for a complex DAW UI, with minimal framework noise.

---

### Option E — SolidJS + Hand-Rolled Design System

**Description**

Use SolidJS, a highly performant fine-grained reactive library with React-like
syntax but without a virtual DOM.

**Pros**

- Very high performance.
- JSX-based with fine-grained reactivity.
- Clean component model.

**Cons**

- Smaller ecosystem and community.
- Less familiar to typical contributors.
- AI tooling support is more limited than for React/Vue/Svelte.
- JSX-centric syntax is less immediately approachable than Svelte’s template
  style for non-front-end specialists.

**Conclusion**

Rejected. Technically attractive, but ecosystem and familiarity concerns make
it a riskier choice than Svelte, with fewer clear human-readability benefits.

---

## Decision

**We will implement Aura using:**

- **Svelte** as the UI framework for the renderer, and
- A **hand-rolled design system** based on:
  - CSS variables for design tokens (colours, spacing, typography, radii, etc.),
  - scoped styles in Svelte components,
  - a shared library of DAW-specific UI primitives
    (faders, meters, clip blocks, headers, toolbars, etc.).

No CSS utility framework (such as Tailwind) will be used in Aura’s core UI.

---

## Rationale

- Svelte yields the **most readable and maintainable** component code, aligning
  with the requirement that any file in Aura should be understandable quickly.
- Svelte’s compiled nature provides excellent performance for complex,
  interactive views without runtime overhead.
- The template syntax is very close to HTML, reducing cognitive load for
  contributors and making components approachable to non-front-end specialists.
- Eliminating hooks and heavy lifecycle abstractions avoids common failure modes
  of large React/TypeScript codebases.
- A hand-rolled design system with CSS variables:
  - supports rich theming (Loophole skins, dark/light themes),
  - keeps styling understandable and centralised,
  - avoids Tailwind’s tendency towards noisy class names and coupling of
    styling to markup.
- While Svelte’s ecosystem is smaller than React’s, the DAW domain benefits
  more from clarity and performance than from maximal library availability,
  especially given the bespoke nature of many Loophole components.

---

## Consequences

### Positive

- Aura’s codebase is likely to remain **approachable and navigable** even as it
  grows large.
- Complex editors (clip, mixer, browser, node graph) can be implemented with
  minimal framework boilerplate.
- The theming model is straightforward and standards-based (CSS variables).
- AI-assisted development remains viable and productive while avoiding the
  worst complexity traps of React + hooks.
- DAW-scale UI performance benefits from Svelte’s compiled output.

### Negative / Trade-Offs

- Some ecosystem integrations may require more custom work than in React
  (e.g. certain component libraries or visualisation tools).
- Contributors familiar only with React or Vue may need to learn Svelte, though
  the learning curve is modest.
- Existing Vue-based patterns from legacy work cannot be copy-pasted directly;
  they must be adapted.

---

## Follow-Up Actions

1. **Create Aura Implementation Plan**
   - Add `docs/plans/aura/implementation.md` describing:
     - project structure (Electron main, Svelte renderer),
     - initial window and IPC wiring,
     - design system skeleton,
     - initial views (project shell, basic mixer/transport shell).

2. **Update Aura Architecture Doc**
   - Update `docs/architecture/b02-aura-architecture.md` to:
     - explicitly name Svelte as the framework,
     - describe the design system and theming strategy,
     - reference the implementation plan.

3. **Update Chorus Overview**
   - Ensure the top-level architecture overview document references Svelte as
     Aura’s UI framework and notes the design system approach.

4. **Create AGENTS File for Aura**
   - Ensure `AGENTS.md` in the Aura repo reflects:
     - Svelte-specific conventions,
     - design system expectations,
     - state and IPC guidelines.

---

## Notes

- This decision is specifically scoped to **Aura’s renderer UI**. Electron as
  the host shell remains the chosen platform.
- If future constraints emerge (e.g. a strong requirement for web-based
  deployment of the UI), this decision may be revisited, but Svelte remains a
  reasonable choice even in that scenario.
