# Pulse History Domain Specification

This document defines the History (undo/redo) domain of the Pulse project
protocol.

It provides a structured, context-aware undo/redo model with:

- a single **global linear history** of transactions,
- **context tags** on transactions (e.g. `mixer`, `timeline`, `routing`),
- **global undo/redo** (all contexts),
- **contextual undo/redo** (e.g. “undo last mixer change”),
- change labels suitable for UI display.

Pulse owns the history model and the application of undo/redo.  
Aura queries history state and triggers undo/redo operations.  
Session reflects a high-level view (e.g. `undoAvailable`, labels).

---

## Contents

- [1. Overview](#1-overview)  
- [2. History Concepts](#2-history-concepts)  
  - [2.1 Transactions](#21-transactions)  
  - [2.2 Contexts](#22-contexts)  
  - [2.3 Global vs Contextual History](#23-global-vs-contextual-history)  
  - [2.4 Non-Undoable Operations](#24-non-undoable-operations)  
  - [2.5 Scripting and Automation](#25-scripting-and-automation)  
- [3. Commands (Aura → Pulse)](#3-commands-aura--pulse)  
  - [3.1 Querying History State](#31-querying-history-state)  
  - [3.2 Global Undo/Redo](#32-global-undoredo)  
  - [3.3 Contextual Undo/Redo](#33-contextual-undoredo)  
  - [3.4 Transaction Navigation](#34-transaction-navigation)  
- [4. Events (Pulse → Aura)](#4-events-pulse--aura)  
  - [4.1 History Stack Events](#41-history-stack-events)  
  - [4.2 Transaction Events](#42-transaction-events)  
- [5. Snapshot Semantics](#5-snapshot-semantics)  
- [6. Relationship to Envelopes, Session and Scripting](#6-relationship-to-envelopes-session-and-scripting)

---

## 1. Overview

The History domain defines:

- how edits are grouped into undoable **transactions**,
- how those transactions are tagged with **contexts**,
- how global and contextual undo/redo operate,
- how history state is exposed to Aura for UI (menus, editor-specific history
  panels),
- how future scripting/automation can integrate with undo.

History does *not* define the low-level state diffing mechanism; it defines:

- the **semantic identity** of history entries,
- their **labels** and **contexts**,
- their **ordering** and **lifetimes**.

---

## 2. History Concepts

### 2.1 Transactions

A **History Transaction** is a logical unit of change:

- identified by a `transactionId`,
- has a human-readable `label` (e.g. “Move Clip”, “Adjust Volume”),
- spans one or more internal operations,
- is either **applied** or **reverted**.

Transactions are ordered in a single linear history:

- `history[0..N-1]`,
- with a **global pointer** indicating the current committed boundary.

New transactions added after undo truncate any “future” transactions (classic
linear undo model).

Internally, transactions may correspond to:

- one or more IPC commands,
- batch operations grouped by Aura (e.g. drag of multiple faders),
- script-driven operations.

The grouping logic is internal to Pulse; the IPC domain only exposes the
result.

---

### 2.2 Contexts

Each transaction may have one or more **contexts**, representing the editing
areas it touched. Examples:

- `mixer` – faders, pans, inserts, sends, Channel config.
- `timeline` – Clips, Lanes, Track structure.
- `routing` – bus routing, Node graph wiring.
- `media` – media pool operations, relinking.
- `timebase` – tempo/signature map edits.
- `global` – high-level project operations (rename, save, etc.).
- `other.*` – future or custom contexts.

Contexts are **tags**, not exclusive categories; a transaction can have:

- `[ "timeline", "mixer" ]` if, for example, an operation adjusts both Clip
  gain and Channel gain.

Contexts enable:

- **contextual history views** in different editors (e.g. Mixer history),
- **contextual undo/redo** commands that operate only on relevant edits.

---

### 2.3 Global vs Contextual History

There is **one global linear history** of transactions.

Global undo/redo:

- traverses the stack in strict chronological order,
- applies or reverts the next/previous transaction, regardless of context.

Contextual undo/redo:

- when asked for context `mixer`, for example:

  - finds the **nearest prior transaction** whose contexts include `mixer`,
  - reverts or reapplies that transaction,

- still moves the global pointer appropriately, so global and contextual
  history remain coherent.

Other transactions *between* contextual steps remain in their current state;
they are **not skipped or reordered**. The underlying model is always a
linear “timeline” of states; contextual commands simply decide which
transaction to target.

---

### 2.4 Non-Undoable Operations

Not all operations should be undoable. Examples:

- transport start/stop,
- UI-only preferences (unless explicitly desired),
- certain diagnostic-only actions.

These operations:

- do not create transactions, or
- are explicitly marked as `undoable: false`.

Envelope-level protocol may include a flag indicating whether commands
contribute to history; the History domain focuses on **what exists** in the
stack, not on how entries are recorded.

---

### 2.5 Scripting and Automation

Future scripting/automation systems will:

- be able to wrap their actions inside transactions,
- optionally define **custom labels** and **contexts**,
- benefit from the same undo/redo semantics as user-driven edits.

The History domain is designed so that scripted operations look identical to
user actions in the history stack, including in contextual views.

---

## 3. Commands (Aura → Pulse)

Aura uses History commands to:

- query history state (global and contextual),
- trigger undo/redo (global and contextual),
- navigate to specific points in history for advanced UIs.

### 3.1 Querying History State

#### **`history.getState`**

Request current history state summary.

Fields (optional filters):

- `includeGlobal` (bool, default `true`),
- `includeContexts` (bool, default `true`),
- optional `contexts` list to filter context-specific summaries
  (e.g. `["mixer", "timeline"]`).

Pulse responds with a state object (via `history.stateUpdated` or a specific
response envelope) including:

- `global`:

  - `canUndo` / `canRedo`,
  - `nextUndo` / `nextRedo` labels and transaction IDs.

- `contexts`:

  - per-context `canUndo` / `canRedo`,
  - per-context `nextUndo` / `nextRedo` labels and transaction IDs.

This state is what Session will mirror as high-level flags.

---

#### **`history.getStack`**

Request a slice of the history stack for display in a “History” panel.

Fields:

- `offset` (int),
- `limit` (int),
- optional `contexts` filter (to show only certain contexts).

Pulse responds with an ordered list of transactions, each including:

- `transactionId`,
- `label`,
- `contexts` array,
- `applied` (bool),
- timestamp.

This is for UI only; navigation is controlled via other commands.

---

### 3.2 Global Undo/Redo

#### **`history.undo`**

Global undo (no context specified).

Fields (optional):

- `count` (default 1) – number of steps to undo.

Pulse:

- reverts the last `count` applied transactions,
- updates the global pointer,
- emits `history.stackChanged`.

---

#### **`history.redo`**

Global redo.

Fields:

- `count` (default 1).

Pulse reapplies up to `count` reverted transactions.

---

### 3.3 Contextual Undo/Redo

#### **`history.undoContext`**

Contextual undo.

Fields:

- `context` (string, e.g. `mixer`, `timeline`),
- `count` (optional; default 1).

Pulse:

- for each step:

  - finds the nearest prior transaction whose `contexts` include `context`,
  - reverts that transaction,

- updates the global pointer accordingly.

If no such transaction exists before the current pointer, no action is
taken for that step.

---

#### **`history.redoContext`**

Contextual redo.

Fields:

- `context`,
- `count` (optional; default 1).

Pulse:

- for each step:

  - finds the nearest future transaction whose `contexts` include `context`,
  - reapplies that transaction, if it is currently reverted,

- updates the global pointer as needed.

---

### 3.4 Transaction Navigation

#### **`history.jumpToTransaction`**

Navigate history to a specific transaction boundary.

Fields:

- `transactionId`.

Pulse:

- computes whether to undo or redo transactions to reach the state where
  `transactionId` is the last applied transaction.
- updates state accordingly.

This is useful for advanced History panels that allow clicking on a specific
point in the stack.

---

## 4. Events (Pulse → Aura)

### 4.1 History Stack Events

#### **`history.stackChanged`**

Emitted whenever the global history stack changes in a way that affects:

- which transactions are applied,
- what undo/redo is available.

Fields may include:

- summary of the new global `canUndo` / `canRedo`,
- `nextUndo` / `nextRedo` labels and IDs.

Aura can re-query detailed state if needed via `history.getState` or
`history.getStack`.

---

#### **`history.transactionAdded`**

A new transaction was added to the history.

Fields:

- `transactionId`,
- `label`,
- `contexts` array.

Useful for incremental updates in UI (e.g. adding one row to a History
panel).

---

### 4.2 Transaction Events

#### **`history.transactionApplied`**

A transaction has been applied (e.g. via redo or jump).

Fields:

- `transactionId`.

---

#### **`history.transactionReverted`**

A transaction has been reverted (e.g. via undo or jump).

Fields:

- `transactionId`.

These events allow contextual UIs (e.g. different editors) to animate or
highlight what changed, if they choose to.

---

## 5. Snapshot Semantics

Project snapshots and History:

- A **snapshot** of project state represents a full, consistent model of the
  project at a point in time (across all domains).
- History is a **sequence of transitions** between such states.

This specification does not require snapshots to be individually stored as
separate project files; they may be efficient internal representations within
Pulse.

Key rules:

- Taking a new snapshot for undo/redo purposes does not necessarily mean
  writing anything to disk beyond what the project persistence model requires.
- Applying a history operation (undo/redo/jump) replaces the current working
  state with the snapshot associated with the target boundary.
- When a new transaction is created after an undo, any “future” transactions
  beyond the current pointer are discarded (standard linear history model).

History state itself (the stack of transactions) may or may not be persisted
across saves; this is a host design decision. The IPC spec assumes that:

- history is valid for the current session,
- after reloading a project, history may start fresh (unless persistent
  history is intentionally implemented).

Session and UI must not assume history survives project reload.

---

## 6. Relationship to Envelopes, Session and Scripting

**Envelopes**

- The envelope specification may include fields like `transactionHint` or
  `undoGroupId`, allowing Aura or scripting engines to group multiple
  commands into a single transaction.
- This History domain assumes that by the time a transaction is visible:

  - grouping is already resolved,
  - label and contexts have been computed.

**Session**

- Session domain reflects a simplified view of history:

  - `undoAvailable` / `redoAvailable`,
  - `nextUndoLabel` / `nextRedoLabel`.

- Session might be updated via `history.stackChanged` and `history.getState`.

**Scripting**

- Scripting environments may:

  - create transactions with custom labels and contexts,
  - call history commands to perform undo/redo,
  - subscribe to history events for automation logic.

The History domain is designed so that scripted and user-driven changes
coexist in a single coherent stack, with clear contextual tagging and robust
global and contextual undo/redo semantics.
