# Chorus Cursor Rules — Architecture

These rules guide Cursor when editing this repository.

## Scope

- This repo contains **no runtime code**.
- Only documentation, specifications, meta-protocol, and task briefs live here.
- Do not add application code, build scripts, or runtime assets.

## When Editing Architecture Docs

- Prefer updating existing documents in `docs/architecture/` rather than
  creating many small, overlapping files.
- Keep sections well-structured, using clear Markdown headings.
- Ensure that Signal, Pulse, Aura, and Chorus responsibilities remain
  consistent with ADRs in `docs/decisions/`.

## When Editing Specs

- Treat files in `docs/specs/` as **authoritative contracts** for other repos.
- Do not change spec formats casually; if a change affects external repos,
  it should be accompanied by:
  - an ADR in `docs/decisions/`, or
  - a note in the relevant task file in `tasks/`.

## Using the Meta-Protocol

- When given a Meta Block (see `docs/meta/meta-protocol.md`), prefer to:
  - Validate it against `docs/meta/meta-schema.json`.
  - Apply exactly the changes described.
- Avoid making additional speculative edits beyond the scope of the Meta Block.

## Tone and Style

- Use British English spelling.
- Aim for clarity and brevity.
- Prefer concrete, implementation-relevant language over marketing language.


