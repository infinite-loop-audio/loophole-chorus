# Cursor Task: Architecture Inbox Append Mode

You are operating in **Architecture Inbox Append Mode**.

Your job is to take any free-form ideas, notes, feature thoughts, half-baked concepts, or speculative architecture discussions provided by the user, and **convert them into structured entries** in:

```
docs/meta/architecture-inbox.md
```

Do *not* overwrite existing content.  
Append new entries to the end of the “Entries” section.

---

## Rules for Processing Input

1. **The user will provide informal ideas.**  
   - They may be short phrases, sentences, or whole paragraphs.  
   - They may be messy, partially formed or ambiguous.

2. **For each idea you detect**, create a structured inbox entry using this format:

```
## <Short Title>
**Tag:** <Engine / Pulse / Aura / IPC / Workflow / DSP / UX / Composer / Misc>  
**Priority:** P1 | P2 | P3  
**Status:** proposed  
A concise, clear summary of the idea, why it matters, and what considerations it raises.
```

3. **Choosing Tags & Priority**
   - Use your best judgement based on the idea’s content.  
   - Default priority is **P3** unless the idea obviously impacts core architecture (then P1 or P2).

4. **Appending Rules**
   - Append entries **just before** the end of the file.  
   - Do not modify or rewrite older entries.  
   - Preserve all existing formatting.

5. **Keep summaries short and precise.**  
   One or two paragraphs maximum.

6. **Do not add trailing newlines** after writing to the file.

---

## What You Should Not Do

- Do not rewrite the file header.
- Do not reorder or delete existing entries.
- Do not combine multiple ideas into one entry unless they are clearly the same concept.
- Do not analyse or question the ideas unless clarification is needed.
- Do not output anything except the updated file when editing.

---

## Workflow Example

**User input:**
> Maybe allow audio lanes to be routed per-clip to different parts of the processor chain?

**Your output (in the file):**
```
## Per-Clip Lane Routing
**Tag:** Pulse / DSP  
**Priority:** P2  
**Status:** proposed  
Allow individual audio lanes within a clip to route to different LaneStream processors in the channel chain. 
Enables advanced sound design workflows and finer control of intra-track signal flow.
```

---

## When User Says “STOP”

If the user says:
- “stop”
- “pause inbox”
- “exit inbox mode”

…then stop modifying the file and await new instructions.

---

## Begin in append mode immediately.

Whenever the user writes informal ideas next, convert and append them.
