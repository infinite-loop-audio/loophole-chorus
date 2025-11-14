# Middle-Agent Command Semantics

This document describes how each `op` value in a Meta Block should be applied
by a middle agent (e.g. Cursor or Codex).

## `create_file`

- **Fields used:** `target`, `content`
- If the file does not exist: create it with `content`.
- If the file already exists: either refuse or overwrite, depending on the
  agent's configuration. Overwriting should be done cautiously.

## `append_section`

- **Fields used:** `target`, `content`
- `target` must point to a Markdown file.
- `content` should contain a complete Markdown section starting with a heading.
- Append `content` at the end of the file, ensuring there is at least one
  blank line before the new section.

## `replace_section`

- **Fields used:** `target`, `section`, `content`
- `target` must point to a Markdown file.
- `section` must match the exact heading line (e.g. `"## Processes and Responsibilities"`).
- The agent should:
  - Find the heading that matches `section`.
  - Replace that heading and all content up to (but not including) the next
    heading of the same or higher level with `content`.

## `delete_section`

- **Fields used:** `target`, `section`
- `target` must point to a Markdown file.
- Behaves like `replace_section`, but removes the section entirely instead of
  replacing it.

## `update_schema`

- **Fields used:** `target`, `content`
- `target` should be a JSON, TypeScript, or similar schema file.
- `content` should contain either:
  - a complete replacement schema, or
  - a clearly delimited fragment with instructions in `notes`.
- For now, agents may treat `content` as a full replacement unless otherwise
  specified.

## `rename_heading`

- **Fields used:** `target`, `section`, `content`
- `target` must be a Markdown file.
- `section` is the current heading line.
- `content` is the new heading line.
- Only the heading line should be changed; the section body remains unchanged.

## Notes for Agents

- All file paths are relative to the root of `loophole-chorus`.
- Agents should avoid performing any changes not directly requested by a Meta
  Block.
- When in doubt, agents should prefer **no change** over guessing.
