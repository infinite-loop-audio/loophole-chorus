# Cursor Task: Normalise Architecture Doc Numbering & Cross-References

You are working in the **loophole-chorus** repository.

This command is **reusable**. Each time it runs, your job is to:

1. Normalise the **sectioned alphanumeric filenames** for architecture docs under `docs/architecture/`.
2. Ensure `docs/architecture/00-index.md` matches the actual files and their ordering.
3. Update all **cross-repo references** *within this repo* that point at architecture docs so they use the current filenames.
4. Produce a short report describing what changed.

Do **not** modify or delete anything under `docs/reports/` except for adding a new report file as described below.

---

## 0. Conventions & Constraints

- Use **British English**.
- Do **not** add trailing newlines to edited files.
- **Do not** modify or delete any files under `docs/reports/` (reports are historical).
- If you need to create a new report, write it to:

  ```text
  docs/reports/YYYY-MM-DD-HHMMSS-architecture-renumber-check.md
  ```

  where `YYYY-MM-DD-HHMMSS` is a full timestamp (24-hour clock, ideally UTC) and the suffix stays exactly `architecture-renumber-check.md`.

- Do not change any non-architecture content except for link targets that reference architecture filenames.

---

## 1. Architecture File Model

Architecture docs live in:

```text
docs/architecture/
```

with filenames following the **section + index + slug** pattern:

```text
<LETTER><TWO_DIGIT_INDEX>-<slug>.md
```

Examples:

- `A01-overview.md`
- `B02-signal-engine.md`
- `I04-stack-nodes.md`

### 1.1 Section letters

Section letters (A, B, C, …) represent logical groups, which are defined by the headings and groupings in:

```text
docs/architecture/00-index.md
```

You must:

- Treat the **index** as the source of truth for:
  - which files belong to which section,
  - the order of files within each section.
- Derive the `LETTER` from the section grouping in the index.
- Derive the `TWO_DIGIT_INDEX` from the **order within that section**.

Do **not** invent your own grouping; always follow the index headings.

---

## 2. Normalise File Names

### 2.1 Scan architecture docs

1. List all `*.md` files in `docs/architecture/` **except**:
   - `00-index.md`
   - any known meta/README files that are explicitly not numbered (if present).

2. For each numbered architecture file:
   - Parse the current filename:
     - If it matches `<LETTER><DIGITS>-<slug>.md`, record:
       - `sectionLetter = LETTER`
       - `indexNumber = DIGITS`
       - `slug`
     - If it is still in an old numeric form like `NN-<slug>.md`, treat it as needing migration.

### 2.2 Reconcile with 00-index.md

Use `docs/architecture/00-index.md` to determine:

- The **section letter** for each entry group.
- The **ordered list of filenames** within each section.

For each section:

1. Build an ordered list of entries from the index under that section.
2. For each entry in order, compute the **expected filename**:

   ```text
   <SECTION_LETTER><TWO_DIGIT_INDEX>-<slug>.md
   ```

   where:
   - `SECTION_LETTER` is the letter for that group (e.g. `A`, `B`, `C`, …).
   - `TWO_DIGIT_INDEX` is `01`, `02`, `03`, … within that section.
   - `slug` is the existing slug part from the current filename (do not change slugs unless they are obviously wrong).

3. Compare expected filename with the actual filename.
   - If they differ, schedule a **rename**:
     - `oldFilename → newFilename`.

Do not reorder sections or entries in the index at this stage; only bring filenames into alignment with the existing ordering.

---

## 3. Apply Renames

For each scheduled rename:

1. Rename the file in `docs/architecture/` from the old name to the new name.
2. Record the mapping in memory for use in link updates and in the final report.

Do **not** touch `00-index.md` yet; update it in the next step.

---

## 4. Update 00-index.md

Open `docs/architecture/00-index.md` and:

1. For each architecture entry that references a filename, update it to match the new `<LETTER><TWO_DIGIT_INDEX>-<slug>.md` name you computed.
2. Preserve:
   - Section ordering,
   - Section headings,
   - Entry titles and descriptions.

You are only changing the **filenames** in the index, not the conceptual ordering.

Ensure that every numbered architecture doc present on disk appears in the index, and vice versa.

---

## 5. Update Cross-References

Search across the **loophole-chorus** repository for references to **old architecture filenames**.

Targets include, but are not limited to:

- Other architecture docs under `docs/architecture/`
- IPC specs under `docs/specs/ipc/`
- Decisions under `docs/decisions/`
- Meta docs under `docs/meta/`
- README files

For each `oldFilename → newFilename` pair:

1. Replace Markdown links that point to the old filename with the new filename.
   - Examples:

     ```markdown
     [Pulse Architecture](../architecture/03-pulse.md)
     ```

     becomes:

     ```markdown
     [Pulse Architecture](../architecture/C01-pulse.md)
     ```

2. Do **not** modify link text (titles) unless they are obviously incorrect.
3. Do **not** change links to other repositories (only intra-Chorus links).

Be careful not to alter bare text mentions that are not links unless they are definitely meant to be file references.

---

## 6. Produce a Report

Create a report file in:

```text
docs/reports/YYYY-MM-DD-HHMMSS-architecture-renumber-check.md
```

The report should summarise:

1. A table of all renames:

   | Section | Old Filename | New Filename |
   |--------|--------------|--------------|
   | B      | 02-signal.md | B01-signal.md |
   | …      | …            | …            |

2. Any index changes made:
   - Which entries were updated.
   - Any mismatches found and resolved.

3. A brief summary of link updates:
   - How many replacements were made.
   - Any files that required manual tweaks.

Do **not** edit or remove older reports.

---

## 7. Final Check

Before finishing:

- Ensure:
  - All architecture files under `docs/architecture/` conform to the `<LETTER><TWO_DIGIT_INDEX>-<slug>.md` scheme (except `00-index.md` and any explicitly unnumbered meta files).
  - `00-index.md` accurately reflects actual filenames and ordering.
  - All internal links in Chorus that pointed at old architecture filenames have been updated.

Do not perform any other refactors or content rewrites as part of this command.
