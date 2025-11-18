# Architecture Refinement Report (2025-11-18 22:19:16)

This report documents targeted architecture refinements applied to align documentation with recent decisions and architectural audit findings.

## Files Modified

### docs/architecture/01-overview.md

- Updated repository table to reflect Pulse as Rust (changed from TypeScript).
- Updated Runtime Components section to clarify Pulse as a standalone Rust server process from the outset, removing references to Pulse being embedded in Aura.
- Updated Pulse description (section 5.2) to explicitly state Pulse is implemented as a separate server process in Rust from the outset, not as a TypeScript layer.
- Removed "Pulse MAY be extracted to its own process" from Evolution Strategy section (now redundant).
- Added new "Security & Privacy" section (section 9) describing trust boundaries, Composer data boundaries, and privacy-first principles for future collaboration features.
- Renumbered subsequent sections (Document Conventions, Summary).

### docs/architecture/02-pulse.md

- Updated Overview section to clarify Pulse is implemented as a separate server process in Rust from the outset, communicating via IPC.
- Added new "Testing & Validation" section (section 13) outlining model-level unit tests, IPC contract tests, scenario tests, and integration tests.
- Renumbered Future Extensions section to section 14.

### docs/architecture/03-signal.md

- Added new "Testing & Validation" section (section 13) outlining graph-level tests, render tests, realtime safety harness, plugin hosting tests, and hardware I/O tests.
- Renumbered Future Extensions section to section 14.

### docs/architecture/05-composer.md

- Added new subsection "Security & Privacy Boundaries" (section 8.1) clarifying:
  - Minimum metadata only policy
  - Opt-in telemetry requirements
  - Advisory role of Composer
  - Offline operation guarantees

### docs/architecture/07-project-versions-and-variants.md

- Added new subsection "Persistence Strategy (High-Level)" (section 5.3) describing:
  - Flexible storage approach (structured file format, embedded database, or hybrid)
  - Required capabilities: atomic writes, crash-safe behaviour, schema version header, migration path
  - SQLite mentioned as a strong candidate without committing to it
  - Note that exact technology will be formalised in a dedicated persistence decision document

### docs/architecture/30-scripting-and-extensibility.md

- Added new subsection "Security Model & Permissions" (section 10.1) describing:
  - Sandboxed execution environment
  - No implicit access to filesystem or network APIs
  - Future permission model plans
  - Architectural constraints on loosening restrictions

### docs/specs/ipc/pulse/plugin-ui.md

- Added new subsection "Plugin UI ID Semantics" (section 2.1) clarifying:
  - `pluginUiId` is always a string identifier in IPC schema
  - Signal may use internal numeric handles but they never cross process boundaries
  - Aura and Pulse must treat `pluginUiId` as opaque string identifiers
  - Examples of string ID format

### docs/specs/ipc/signal/plugin-ui.md

- Updated "Plugin UI ID" section (section 2.2) to clarify:
  - `pluginUiId` is always a string identifier in IPC schema
  - Signal may use internal numeric handles but they never cross process boundaries
  - Aura and Pulse must treat `pluginUiId` as opaque string identifiers

## Summary

All changes align architecture documentation with:
- The decision that Pulse is a Rust server process from the outset (not TypeScript)
- Clear security and privacy boundaries across components
- Explicit testing strategies as part of the architecture
- Flexible but well-defined persistence strategy
- Consistent IPC semantics for plugin UI identifiers

The documentation now clearly reflects the current architectural decisions while maintaining flexibility for future implementation choices.

