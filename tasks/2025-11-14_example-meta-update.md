# Task: Example Use of the Meta-Protocol

This is an example task file illustrating how Meta Blocks can be used in
practice to update documentation in Chorus.

## Context

We want to update the architecture overview to clarify that Pulse is initially
hosted inside Aura, but may become its own process later.

## Meta Block

```json
{
  "op": "replace_section",
  "target": "docs/architecture/01-overview.md",
  "section": "## Processes and Responsibilities",
  "content": "## Processes and Responsibilities\\n\\n(Updated section text goes here...)",
  "notes": "Clarify Pulse hosting and future extraction."
}
```

Instructions for Middle Agents
	1.	Validate the Meta Block against docs/meta/meta-schema.json.
	2.	Apply the described change to docs/architecture/01-overview.md.
	3.	Do not modify any other sections of the file.
	4.	Leave a clear git commit message, e.g.:
	•	chore(docs): clarify Pulse hosting in architecture overview
