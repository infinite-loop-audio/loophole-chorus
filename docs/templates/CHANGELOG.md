# Changelog

All notable changes to this project will be documented in this file.

## Format

- One entry per line.
- Strict chronological order: **newest at the top**.
- Each entry must include:
  - **Timestamp** (UTC, ISO 8601): `YYYY-MM-DD HH:MM:SS UTC`
  - **Tag** in square brackets:
    - `[added]` – new features, new files
    - `[changed]` – behaviour, APIs, refactors
    - `[fixed]` – bugs, crashes, shutdown issues
    - `[removed]` – deletions, deprecations
    - `[docs]` – documentation or specification updates
    - `[dev]` – build tools, tests, config, CI
  - Free-text description (British English)

### Example

```
(2025-11-21 22:46:10 UTC) [changed] Normalised IPC event naming model (responses removed, events unified).
(2025-11-21 22:42:55 UTC) [dev] Reorganised Aura renderer domain structure.
(2025-11-21 21:10:13 UTC) [fixed] Corrected Pulse shutdown sequencing to avoid SIGTERM during graceful quit.
```

---

## [Unreleased]

(YYYY-MM-DD HH:MM:SS UTC) [added] …

---

## [0.1.0] – YYYY-MM-DD
Initial baseline release.
