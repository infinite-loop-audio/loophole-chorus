# Collaboration Architecture

This document defines the **Collaboration Architecture** for Loophole.

It outlines how multiple users can work together on a project, either:

- **asynchronously** (passing project files around, version histories, offline changes), or  
- **synchronously** (real-time shared editing, multiple users controlling the same session).

The goal is not to specify implementation in full detail today, but rather to
make Loophole’s core systems **collaboration-ready** from the architecture
onward. Future releases can then build collaboration progressively without
retrofit or architectural disruption.

It complements:

- `02-pulse.md` (Pulse architecture)  
- `03-signal.md` (Signal engine architecture)  
- `04-aura.md` (UI and UX architecture)  
- `06-tracks-channels-and-lanes.md`  
- `07-clips.md`  
- `08-parameters.md`  
- `20-rendering-and-offline-processing.md`  
- `26-ux-and-visual-layer.md`  
- `33-plugin-library-and-browser.md`  
- Decision records related to IPC and Pulse process model

This document does **not** mandate any full collaboration implementation for v1,
but it ensures the foundations are correct so that Loophole can evolve into a
collaboration-first DAW without major architectural upheaval.

---

# 1. Goals

Loophole’s Collaboration Architecture has four core goals:

## 1. **Be opt-in and incremental**

Local-only workflows remain untouched.  
Collaboration layers **sit on top** of project storage and Pulse’s model.

## 2. **Provide both synchronous and asynchronous models**

- Asynchronous:  
  Users pass project revisions, merge changes, resolve conflicts, and branch
  project timelines.

- Synchronous:  
  Multiple users edit the same project simultaneously, with real-time
  synchronisation and presence indicators.

## 3. **Minimise coupling with Signal**

Signal is a machine-local audio engine.  
Collaboration must:

- leave Signal isolated,  
- never stream audio over the collaboration layer,  
- replicate decisions/events rather than audio.

## 4. **Allow fine-grained, conflict-aware editing**

- Parameter changes,
- Clip edits,
- NodeGraph modifications,
- Plugin operations,
- Mixer adjustments.

Every edit should be serialised in a way that:

- can be diffed,
- can be merged,
- can be timestamped,
- and can be rejected/reapplied as needed.

This depends on Pulse being the source of truth.

---

# 2. Collaboration Models

## 2.1 Asynchronous Collaboration

This includes:

- version histories,
- branches,
- offline editing,
- merging revisions,
- project snapshots that track ancestry.

Pulse model changes can be represented as:

```
EditOperation {
  operationId;
  timestamp;
  userId;
  domain: string;          // tracks, clips, routing, etc.
  command: string;         // e.g. "clip.split", "node.move"
  payload: object;
  baseVersion: VersionId;
}
```

These operations allow:

- inverted diffs (for undo),
- merge conflict detection,
- efficient project histories,
- integration with cloud storage services.

Asynchronous collaboration depends on:

- document versioning (architecture doc forthcoming),
- Pulse-level undo/redo primitives.

## 2.2 Synchronous Collaboration

Real-time shared editing requires:

- a shared Pulse instance (remote-capable),  
- or Pulse instances that synchronise state via a collaboration server.

The architecture supports two models:

### Model A — **Shared Pulse Server**

One Pulse instance runs in a network-accessible mode.  
Multiple Aura clients connect to it.

- Pros: One authoritative model, not distributed.  
- Cons: Requires hosting and good network.

### Model B — **Distributed Pulse Instances with CRDT-like Synchronisation**

Each user runs Pulse locally.  
Changes are synchronised via:

- CRDT-inspired domain-specific data types,  
- operational transforms built on EditOperations,  
- a lightweight collaboration coordinator.

This model helps support:

- high-latency networks,
- intermittent connectivity,
- offline-capable collaboration.

Both models are compatible with the architecture.  
This document leans toward A for early versions, with B as the long-term goal.

---

# 3. Role of Each Layer

## 3.1 Aura

Aura:

- presents collaboration UIs:
  - who is editing what,
  - live cursors,
  - clip/parameter highlighting,
  - presence indicators;
- sends user-intent commands as normal IPC to Pulse;
- displays merge conflicts and resolution tools;
- takes care of user identity in the UI layer.

Aura never performs merge logic; it shows Pulse-computed diffs and history.

---

## 3.2 Pulse

Pulse is the **collaboration brain**.

Pulse handles:

- authoritative project state (local or remote),
- sequencing of edit operations,
- version metadata (commit IDs, user IDs, timestamps),
- conflict detection,
- conflict resolution strategies,
- diff generation for clients,
- branching and merging,
- collaboration session management.

Critically, Pulse maintains:

### **Collaboration Clock**

A logical timestamping system ensuring:

- causality ordering,
- multi-user compatible edit sequencing,
- stable ordering for automation ramps and parameter gesture sessions.

Pulse already supports the idea of modelling changes as high-level operations;
collaboration formalises this into a serialised timeline.

---

## 3.3 Signal

Signal is intentionally **not** network-aware.

Signal receives:

- only fully resolved project commands,
- resulting from Pulse reconciling collaboration events.

Signal never:

- communicates directly with remote clients,
- participates in versioning,
- propagates edits.

Signal remains a purely local engine.

---

# 4. Project State & Versioning

A full document on versioning is forthcoming, but collaboration relies on:

## 4.1 Project Version IDs

Each committed state has:

```
Version {
  versionId;
  parentVersionId;
  timestamp;
  userId;
}
```

This allows:

- branching,
- history graphs,
- rebase/merge operations,
- time-travel on the project.

## 4.2 Edit Operations

Each user action is an `EditOperation` applied to the model.

Pulse serialises all edit operations:

- into local history,
- into collaboration messages (synchronous),
- into patch files (asynchronous).

## 4.3 Merging & Conflict Resolution

Examples of conflicts:

- Two users move the same Clip simultaneously.
- One user deletes a Node another is modifying.
- Two users adjust the same plugin parameter.

Pulse supports:

### **Resolution strategies**

- Last-writer-wins (LWw) for trivial cases,
- Domain-specific merging for structured data:
  - merging automation curves,
  - merging timeline edits,
  - merging NodeGraph edits,
- Explicit conflict surfaces for UI review:
  - Aura displays the diffs,
  - user selects which version wins,
  - Pulse applies resolution and replays dependent operations.

---

# 5. Presence & Awareness

Synchronous collaboration includes:

- user cursors in piano roll, arranger and mixers,
- user names/identities associated with edits,
- clip highlights indicating current editor,
- “user is auditioning” feedback (non-destructive),
- “pending edits” notifications.

Pulse manages presence via a collaboration session:

```
PresenceInfo {
  userId;
  displayName;
  activity: "editing clip" | "moving node" | …
  selection?: ClipId | NodeId | TrackId | Range;
}
```

Aura renders this contextually.

---

# 6. Remote Assets & Limitations

Media files referenced by the project must be:

### a) local,  
or  
### b) synchronised via external file-sync tools (Dropbox, iCloud Drive),  
or  
### c) referenced via a Loophole collaboration server that streams project media.

The Collaboration Architecture is careful to:

- allow offline work,
- allow missing media placeholders,
- not block project operation when media is missing,
- maintain correct media mapping so future users can relink safely.

Signal loads audio/video streams only from local paths; collaboration must ensure
the user has those paths available.

---

# 7. Security & Privacy

To support remote collaboration:

- Pulse must authenticate user identity for session membership.
- Collaboration messages must be encrypted when routed over networks.
- No personal data is embedded in the project file unless required for attribution.
- Audio and MIDI data are only shared when users explicitly choose to.

Composer’s data-sharing is **separate** from collaboration data.

---

# 8. IPC Considerations

Collaboration requires defining new IPC domains (Pulse ↔ Aura).  
This is not required immediately but the architecture must allow it.

### Aura → Pulse

- `collab.joinSession`
- `collab.leaveSession`
- `collab.publishEdit`
- `collab.requestHistory`
- `collab.requestPresence`

### Pulse → Aura

- `collab.presenceUpdate`
- `collab.editApplied`
- `collab.remoteEdit`
- `collab.history`
- `collab.conflict`
- `collab.resolutionRequired`

Internal Pulse ↔ Pulse collaboration flows depend on the model (centralised vs CRDT-inspired).

Signal has **no collaboration IPC**.

---

# 9. Future Extensions

1. **Live performance collaboration**  
   Multiple users controlling parts of the same project in real-time performance.

2. **Version branches for experimentation**  
   One user can switch to an alternate version/branch without affecting others.

3. **Collaborative instrument editing**  
   Multi-user sampler programme editing, with conflict-aware region changes.

4. **Locking mechanisms**  
   Allow optional “soft locks” on tracks, clips, nodes or ranges.

5. **Ghost playback states**  
   Each user can audition different takes or scenes without affecting the shared transport.

6. **Split roles**  
   - Producer mode,
   - Performer mode,
   - Mixing engineer mode,
   each with different permissions.

---

# 10. Summary

The Collaboration Architecture:

- builds collaboration on top of Pulse, not Signal,
- integrates asynchronous versioning and synchronous live editing,
- uses serialised edit operations as the atomic unit of change,
- ensures deterministic conflict resolution pathways,
- provides user presence and awareness tools,
- keeps Signal isolated from network complexity,
- and defines a scalable foundation for future collaborative workflows.

With this architecture in place, Loophole can grow from a local-first DAW into a
powerful collaborative production environment without rewriting its core.
