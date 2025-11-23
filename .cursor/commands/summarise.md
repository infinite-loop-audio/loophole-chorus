# summarise-repo

## Usage
Run this command with the repo name as the first argument:

```
/summarise-repo aura
/summarise-repo pulse
/summarise-repo chorus
/summarise-repo signal
```

The repo name is only used for context in the final report (e.g., “State of Aura Repo”), but the logic works identically for all repositories.

---

## Goal
Produce a clear, structured summary of the **current repository**, including:

- **Directory tree** (collapsed, clean, readable)
- **Folder purpose descriptions**
- **Architecture alignment** with Chorus specs & ADRs
- **IPC consistency** (envelope fields, naming, semantics)
- **Cross-repo consistency checks**
- **Code quality issues** & deviations from AGENTS rules
- **A prioritised next-step list**

This command helps keep ChatGPT fully up-to-date with any repo at any time.

---

## Output Format

### 1. Title
```
## Repository Summary: <REPO_NAME>
```

### 2. Directory Structure
- Provide a tree view of the repo.
- **Exclude**:
  - node_modules
  - dist / build artefacts
  - target
  - .git
  - .DS_Store
  - any cache folders

### 3. Folder Purposes
For each top-level folder:
- Explain its purpose
- Note whether the structure is clean & matches expectations for the repo (Pulse/Aura/Signal/Chorus)

### 4. IPC & Spec Alignment
Check against Chorus:
- Envelope fields: `domain`, `name`, `kind`, `priority`, correlation IDs
- Naming conventions per ADR-011
- Command/event symmetry per ADR-012
- Domain names match Chorus’ canonical list
- No redundant domain prefixes in names
- Snapshot usage & handshake rules (if relevant)
- Any feature drift or missing domains

### 5. Code Quality & Consistency
Check:
- Naming conventions (camelCase, kebab-case, snake_case depending on language)
- Function formatting rules (multi-line params, spacing rules)
- Flow-statement spacing rules
- Consistency with AGENTS.md
- Dead / unreachable modules
- Test coverage & structure
- Whether architecture boundaries are respected

### 6. Cross-Repo Consistency
Check for mismatches with other Loophole repos:
- IPC fields
- Domain names
- Event/command shapes
- Envelope semantics
- Shared identity rules
- Spec drift between codebases

### 7. Suggested Improvements / Next Steps
Provide a short, ordered list:
- Highest priority first
- Mix of architecture, cleanup, and bugs
- Each item max one line

---

## Rules
- **Do not modify any files.**
- **Do not generate code** — analysis only.
- Keep output concise but thorough.
- Use British English.
- Use accurate file paths.
- Respect repo conventions for naming, formatting, and architecture.
