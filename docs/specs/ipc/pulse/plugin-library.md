# Pulse Plugin Library Domain Specification

This document defines the **Plugin Library** domain of the Pulse IPC protocol.

The Plugin Library domain covers user-facing **plugin organisation and browsing**
concerns that sit on top of the engine-facing plugin lifecycle:

- plugin discovery surface,
- library search,
- deduplication of multi-format variants,
- user renaming and tagging,
- category assignment,
- favourites and hiding,
- plugin chains and templates,
- plugin presets (engine-agnostic view),
- replacement and variant selection.

It does *not* deal with engine-level processing or sandboxing; those are defined in:

- `docs/specs/ipc/pulse/plugin-ui.md` (plugin UI window management)
- `docs/specs/ipc/signal/plugin.md` (plugin lifecycle and sandboxing)
- `docs/specs/ipc/signal/engine.md` (engine-level operations)

For the architectural overview of the Plugin Library system, see:

- `docs/architecture/33-plugin-library-and-browser.md`

The Plugin Library domain is **Aura ↔ Pulse only**. Pulse may in turn call
existing Pulse → Signal plugin lifecycle commands when insert/replace actions
are requested, but those are out of scope for this document.

---

## 1. Responsibilities & Scope

The Plugin Library domain:

- exposes the **canonical list of plugins** available to Loophole,
- aggregates metadata from:
  - Signal scanning results,
  - Composer knowledge,
  - user overrides (names, tags, categories, visibility, favourites),
- presents a **deduplicated multi-format view** of plugins,
- allows Aura to:
  - search and filter plugins,
  - rename and retag plugins,
  - mark favourites,
  - hide/restore variants,
  - choose preferred format variants,
  - manage user chains and presets,
  - request insert/replace operations in the NodeGraph.

The Plugin Library domain **never**:

- talks directly to the engine audio threads,
- loads plugin binaries,
- performs DSP.

It is purely a **model and coordination layer** for UI, backed by Pulse’s
persistent library state.

---

## 2. Data Model (Conceptual)

The IPC layer does not expose the full internal structures, but the behaviour
assumes a conceptual model:

- **PluginEntry**
  - A user-facing plugin identity (e.g. “FabFilter Pro-Q 3”),
  - Aggregates all platform/format variants.

- **PluginVariant**
  - Single binary (VST3, CLAP, AU),
  - Path, format, version, sandbox policy.

- **UserOverrides**
  - User-defined visible name,
  - tags and categories,
  - hidden flag,
  - favourite flag,
  - preferred variant.

- **PluginChain**
  - A reusable chain of Nodes using specific plugins and parameter sets.

- **PluginPreset**
  - A reusable state snapshot for a specific plugin.

All commands/events in this domain manipulate these conceptual entities.

---

## 3. Commands (Aura → Pulse)

### 3.1 Library Introspection

**`pluginLibrary.requestSummary`**  
Request a summary of the current library state.

- No payload.
- Pulse responds with `pluginLibrary.summary`.

Used when the browser first opens, or when Aura reconnects.

---

**`pluginLibrary.search`**  
Search the plugin library by term and optional filters.

Payload fields (illustrative):

- `query: string` – free text; may be empty for “show all”.
- `filters?: object`
  - `tags?: string[]`
  - `categories?: string[]`
  - `manufacturer?: string[]`
  - `format?: ("vst3"|"clap"|"au")[]`
  - `features?: object` (e.g. `{ sidechain: true, ara: false }`)

Pulse responds with `pluginLibrary.searchResults`.

---

**`pluginLibrary.requestDetails`**  
Request detailed information for one or more PluginEntries.

Payload fields:

- `pluginIds: string[]` – stable plugin IDs.

Pulse responds with `pluginLibrary.details`.

---

### 3.2 User Overrides (Names, Tags, Categories)

**`pluginLibrary.renamePlugin`**  
Change the user-visible name of a plugin entry.

Payload fields:

- `pluginId: string`
- `visibleName: string|null`
  - `null` to clear the override and revert to default name.

Triggers `pluginLibrary.entryUpdated`.

---

**`pluginLibrary.addTag`**  
Add one or more tags to a plugin.

Payload fields:

- `pluginId: string`
- `tags: string[]`

Pulse merges tags into the plugin’s tag set and emits `pluginLibrary.entryUpdated`.

---

**`pluginLibrary.removeTag`**  
Remove tags from a plugin.

Payload fields:

- `pluginId: string`
- `tags: string[]`

Triggers `pluginLibrary.entryUpdated`.

---

**`pluginLibrary.assignCategories`**  
Replace the set of user-assigned categories for a plugin.

Payload fields:

- `pluginId: string`
- `categories: string[]` – category IDs or names (Pulse resolves).

Triggers `pluginLibrary.entryUpdated`.

---

### 3.3 Visibility & Favourites

**`pluginLibrary.setFavourite`**  
Mark or unmark a plugin as a favourite.

Payload fields:

- `pluginId: string`
- `favourite: boolean`

Triggers `pluginLibrary.entryUpdated`.

---

**`pluginLibrary.setHidden`**  
Hide or unhide a plugin from normal browser listings.

Payload fields:

- `pluginId: string`
- `hidden: boolean`

Hidden plugins:
- remain available for project loading,
- do not appear in normal listing unless explicitly requested.

Triggers `pluginLibrary.entryUpdated`.

---

### 3.4 Variant Selection

**`pluginLibrary.selectVariant`**  
Explicitly choose which variant (format) to prefer for a plugin.

Payload fields:

- `pluginId: string`
- `variantId: string|null`
  - `null` to revert to automatic selection.

Pulse updates internal preference and emits `pluginLibrary.entryUpdated`.

---

### 3.5 Chains & Templates

**`pluginLibrary.createChain`**  
Create a new plugin chain definition.

Payload fields (high-level):

- `name: string`
- `description?: string`
- `nodes: ChainNodeDescriptor[]`
- `tags?: string[]`

Pulse assigns a `chainId` and emits `pluginLibrary.chainAdded`.

---

**`pluginLibrary.updateChain`**  
Modify an existing chain.

Payload fields:

- `chainId: string`
- `name?: string`
- `description?: string`
- `nodes?: ChainNodeDescriptor[]`
- `tags?: string[]`

Triggers `pluginLibrary.chainUpdated`.

---

**`pluginLibrary.deleteChain`**  
Remove a chain definition.

Payload fields:

- `chainId: string`

Triggers `pluginLibrary.chainRemoved`.

---

**`pluginLibrary.requestChains`**  
Request the list of available chains (optionally filtered).

Payload fields (optional):

- `filters?: object`
  - `tags?: string[]`
  - `containsPluginId?: string`

Pulse responds with `pluginLibrary.chains`.

---

### 3.6 Presets (Library-Level)

Note: This domain covers library-level presets (user/factory/project scope).  
Engine-level state application remains in the plugin lifecycle domain.

**`pluginLibrary.savePreset`**  
Save a preset for a given plugin.

Payload fields:

- `pluginId: string`
- `presetName: string`
- `scope: "user" | "project"`
- `tags?: string[]`
- `overwriteIfExists?: boolean`
- `sourceInstanceId?: string` – optional plugin instance to read state from.

Pulse:
- pulls the plugin state (via plugin lifecycle IPC),
- stores preset metadata,
- emits `pluginLibrary.presetAdded`.

---

**`pluginLibrary.deletePreset`**  
Delete a preset.

Payload fields:

- `presetId: string`

Triggers `pluginLibrary.presetRemoved`.

---

**`pluginLibrary.requestPresets`**  
Request presets for a given plugin.

Payload fields:

- `pluginId: string`
- `scopeFilter?: ("factory"|"user"|"project")[]`

Pulse responds with `pluginLibrary.presets`.

---

### 3.7 Insert & Replace (High-Level Library Commands)

These commands are syntactic sugar that combine:

- library lookup,
- NodeGraph manipulation,
- plugin lifecycle calls.

The detailed NodeGraph/plugin IPC is handled by other domains.

**`pluginLibrary.insertPlugin`**  
Request insertion of a plugin at a given context location.

Payload fields (illustrative):

- `pluginId: string`
- `context: object`
  - `channelId?: string`
  - `nodeId?: string`       – insert after/before this node
  - `position?: "start"|"end"|"replace"`  
- `initialPresetId?: string`
- `smartRouting?: boolean`  – allow Pulse to use heuristic insert behaviour

Pulse:
- resolves the plugin variant,
- decides exact graph changes,
- dispatches NodeGraph + plugin lifecycle commands,
- emits Node/Channel events as usual.

Aura does **not** need to know the low-level details.

---

**`pluginLibrary.replacePlugin`**  
Replace one plugin instance with another, preserving state where possible.

Payload fields:

- `nodeId: string`        – the Node to replace
- `newPluginId: string`
- `strategy?: "bestEffort"|"parametersOnly"|"rawStateOnly"`

Pulse:
- looks up old and new plugin metadata,
- attempts parameter/state mapping,
- applies mapped state via plugin lifecycle IPC,
- emits `pluginLibrary.replacementResult`.

---

## 4. Events (Pulse → Aura)

### 4.1 Library State

**`pluginLibrary.summary`**  
Sent in response to `pluginLibrary.requestSummary`.

Includes:
- a minimal summary of all PluginEntries:
  - `pluginId`
  - `visibleName`
  - `manufacturer`
  - `favourite`
  - `hidden`
  - `primaryTags`
  - `formats` summary

Designed for initial population of browser UI.

---

**`pluginLibrary.searchResults`**  
Search results for `pluginLibrary.search`.

Payload fields:

- `query: string`
- `results: PluginEntrySummary[]`  
- `suggestions?: object` (Composer hints, spelling, similar terms)

---

**`pluginLibrary.details`**  
Detailed information about one or more PluginEntries.

Payload fields:

- `entries: PluginEntryDetail[]`

Includes:
- full tag list,
- categories,
- variants,
- composerHints (if present),
- parameter summary (if needed).

---

### 4.2 Entry Updates

**`pluginLibrary.entryUpdated`**  
Emitted when any of the following changes:

- visible name,
- tags,
- categories,
- favourite flag,
- hidden flag,
- preferred variant.

Payload fields:

- `pluginId: string`
- `changedFields: string[]`  
- `entry?: PluginEntrySummary` (optional updated summary)

---

**`pluginLibrary.libraryUpdated`**  
Broad notification that the library has changed more extensively:

- new plugins discovered,
- plugins removed,
- composer metadata sync.

Aura should respond by refreshing views as appropriate.

---

### 4.3 Chains

**`pluginLibrary.chainAdded`**  
New chain created.

Payload fields:

- `chain: ChainSummary`

---

**`pluginLibrary.chainUpdated`**  
Existing chain updated.

Payload fields:

- `chain: ChainSummary`

---

**`pluginLibrary.chainRemoved`**  
Chain removed.

Payload fields:

- `chainId: string`

---

**`pluginLibrary.chains`**  
Response to `pluginLibrary.requestChains`.

Payload fields:

- `chains: ChainSummary[]`

---

### 4.4 Presets

**`pluginLibrary.presetAdded`**  
New preset stored.

Payload fields:

- `preset: PresetSummary`

---

**`pluginLibrary.presetRemoved`**  
Preset deleted.

Payload fields:

- `presetId: string`

---

**`pluginLibrary.presets`**  
Response to `pluginLibrary.requestPresets`.

Payload fields:

- `pluginId: string`
- `presets: PresetSummary[]`

---

### 4.5 Replacement & Missing Plugins

**`pluginLibrary.missingPlugin`**  
Indicates that a plugin referenced by a project is not currently available.

Payload fields:

- `pluginId: string`
- `pluginName: string`
- `usedBy: NodeReference[]`
- `possibleReplacements?: ReplacementSuggestion[]`

---

**`pluginLibrary.replacementResult`**  
Result of a `pluginLibrary.replacePlugin` request.

Payload fields:

- `nodeId: string`
- `oldPluginId: string`
- `newPluginId: string`
- `success: boolean`
- `details?: string`
- `mappingInfo?: object` (optional diagnostics about mapped parameters)

---

## 5. Error Handling

The Plugin Library domain follows the general IPC error semantics:

- For invalid plugin IDs or chains:
  - Pulse emits a domain-specific error event (e.g. `pluginLibrary.error`),
  - or includes an `error` field in the command response event.

Typical error cases:

- unknown or stale `pluginId`,
- preset not found,
- chain node uses a plugin no longer available,
- replacement mapping failed.

Error events should be non-fatal and designed for UI-level feedback.

---

## 6. Versioning & Evolution

The Plugin Library domain is expected to evolve alongside:

- plugin lifecycle changes,
- Composer capabilities,
- future store/marketplace integrations.

Versioning considerations:

- Each command/event may gain optional fields over time.
- Backwards compatibility should be maintained wherever possible.
- New search filters and metadata attributes can be introduced without breaking existing clients.

Aura should be defensive:
- ignore unknown fields gracefully,
- rely only on documented required fields.

---

## 7. Summary

The Plugin Library domain:

- provides a **deduplicated, user-friendly view** of the plugin ecosystem,
- exposes rich organisational tools (rename, tags, categories, favourites, hiding),
- allows for complex chains and presets,
- supports high-level insert and replace workflows,
- integrates Composer’s global intelligence,
- maintains a clear separation from engine-level plugin lifecycle.

Together with the plugin lifecycle and sandboxing architecture, this domain
forms the backbone of Loophole’s plugin management experience, finally solving
long-standing pain points in how DAWs organise and present plugins.
