# Plugin Library & Browser Architecture

This document defines the **plugin library, browser and organisational model**
for Loophole. It focuses on how plugins are:

- discovered,
- categorised,
- renamed,
- deduplicated,
- searched,
- tagged,
- organised,
- favourited,
- inserted into Track/Channel NodeGraphs,
- migrated between formats or versions.

This architecture provides the **user-facing counterpart** to the engine-facing
plugin lifecycle defined in:

- `32-plugin-lifecycle-and-sandboxing.md`

It integrates with:

- Composer metadata knowledge (`05-composer.md`)
- Media Browser (`13-media-architecture.md`)
  (but remains a distinct system)
- NodeGraph architecture (`11-node-graph.md`)
- Editor and UX layers (`26-ux-and-visual-layer.md`)
- Pulse plugin metadata domain (parameter aliasing, identity, replacements)

The Plugin Library is a **first-class citizen** in Loophole, designed to fix
long-standing issues in other DAWs around plugin management.

---

# 1. Goals

1. **Make plugin browsing effortless and expressive**
   - Find what you need immediately.
   - Intelligent search, tags, categories.

2. **Remove plugin duplication noise**
   - Hide redundant VST3/CLAP/AU duplicates.
   - Automatically group variants of the same plugin.

3. **Support user-defined naming and organisation**
   - User can rename plugins without affecting project integrity.
   - Create custom categories ("Synths", "Warmers", "Mastering", etc.).

4. **Leverage Composer for metadata**
   - Learn global categorisation patterns.
   - Provide suggested tags.
   - Assist with plugin aliasing/matching across versions.

5. **Provide high-speed search**
   - By name, tag, manufacturer.
   - By plugin features (e.g., “sidechain”, “multiband”, “saturator”).
   - By parameter names (advanced).

6. **Allow drag-from-browser → drop-onto-channel/nodegraph**
   - Intuitive placement in the mixer or node chain.

7. **Support plugin chains and templates**
   - User-defined or Composer-suggested chains.
   - “Starter racks” or favourite chains.

8. **Ensure plugin identity stability**
   - Renames do *not* break projects.
   - Hidden plugins still load existing projects properly.

9. **Handle plugin absence gracefully**
   - Show missing plugin states with replacement suggestions.

---

# 2. Plugin Library Conceptual Model

Pulse maintains a canonical representation:

```
PluginLibrary {
  plugins: PluginEntry[];
  categories: Category[];
  userOverrides: UserOverrides;
  composerHints: ComposerHints;
}
```

### 2.1 PluginEntry

```
PluginEntry {
  pluginId;                   // stable ID from PluginMeta
  visibleName;                // user-renamable display name
  formalName;                 // from plugin metadata
  manufacturer;
  tags: string[];
  categories: CategoryId[];
  variants: PluginVariant[];  // VST3/CLAP/AU variants
  favourite: bool;
  hidden: bool;               // user-hidden duplicates
}
```

### 2.2 PluginVariant

```
PluginVariant {
  format: vst3 | clap | au;
  path;
  version;
  sandboxPolicy;
  active;                     // whether variant is selected for instantiation
}
```

Pulse chooses a **default active variant**.  
Users may override this (e.g., “prefer CLAP when available”).

---

# 3. Plugin Deduplication

Most DAWs show:

- plugin.vst3  
- plugin.clap  
- plugin.au  

…as separate entries, causing huge clutter.

Loophole groups them under *one* PluginEntry.

## 3.1 Grouping Rules

Variants are grouped when:

- internal vendor & plugin name match, OR
- Composer reports a high-confidence match, OR
- Pulse finds matching metadata (IO config, parameter set), OR
- the user manually merges entries.

## 3.2 Default Variant Selection

Pulse chooses the best variant based on:

1. **Format preference order**  
   (configurable, default: CLAP → VST3 → AU)

2. **Sandbox support**

3. **Feature completeness**

4. **Composer global recommendations**

5. **Stability history**

Users may override globally or per plugin.

---

# 4. Renaming, Tagging & Organisation

## 4.1 User Renaming

User renames the plugin:

> “FabFilter Pro-Q 3” → “Master EQ”  
> “Surge XT” → “Dreamsynth”

These changes:
- never affect the underlying pluginId,
- never break existing projects,
- are stored in `userOverrides`,
- are synced across sessions on the same machine.

## 4.2 Plugin Tags

Tags provide high-level and searchable descriptors:

- dynamics  
- mastering  
- saturator  
- modulation  
- synth  
- drum  
- utility  
- creative  
- character  
- modelling  
- MPE-capable  
- ARA-capable  

Users may:
- add/remove tags,
- create custom tags (“Chewy”, “Warm”, “Harsh”).

Composer suggests statistically common tags for each plugin.

## 4.3 Categories

Categories are user-facing groups, different to tags.

Examples:
- *Dynamics*
- *EQ*
- *Reverb*
- *FX*
- *Synths*
- *Mix Tools*
- *Mastering*

Categories behave like folders in the browser.

Plugins may belong to multiple categories.

---

# 5. Browser Structure (Aura)

The Plugin Browser appears in:

- Side panel,
- Floating panel,
- Dedicated full-screen view,
- Popup search (“Quick Insert”).

### 5.1 Sections

```
• Recent
• Favourites
• Categories
    • Dynamics
    • EQ
    • Synth
    • …
• Tags
• Manufacturers
• All Plugins (alphabetical)
```

### 5.2 Plugin Cards

Each plugin is shown as a “card” with:
- display name,
- tags,
- manufacturer,
- variant icon,
- favourite star,
- preview affordance.

Hover shows:
- parameters summary,
- version,
- supported formats,
- Composer hints,
- plugin stability score.

---

# 6. Search System

The search engine allows:

### 6.1 Basic Search
- name or part of name,
- manufacturer,
- format.

### 6.2 Tag Search
e.g.,
```
tag:mastering
tag:synth
tag:mpe
```

### 6.3 Parameter Search
Find plugins with specific parameters:

```
param:cutoff
param:self-oscillation
param:multiband
```

### 6.4 Feature Search

```
sidechain:true
ara:true
oversampling:true
latency>10ms
deterministic:false
```

### 6.5 Composer-Semantic Search

Composer-assisted “conceptual” searches:

```
“warm saturator”
“lofi texture”
“wide reverb”
“fm bass maker”
“analog compressor”
```

Composer maps these to statistically common plugin/tag combinations.

---

# 7. Plugin Insertion Workflows

### 7.1 Drag → Drop

Users drag a plugin from the browser:

- onto a channel strip → inserts plugin,
- into a specific slot on the NodeGraph,
- onto a Lane → creates instrument node,
- onto a track header → auto-create channel if needed.

### 7.2 Quick Insert (Cmd+P style)

Popup search → Enter → inserts plugin at:
- selected node position,
- top of chain,
- or following current selection.

### 7.3 Double-Click

- If a Channel is selected → insert at top.
- If Node is selected → insert after.

### 7.4 Intelligent Insert

Option for “smart insertion”:
- if plugin is a reverb → propose FX bus insertion,
- if plugin is an EQ → insert before compression,
- if plugin is generator → replace instrument slot.

Composer can contribute heuristics.

---

# 8. Plugin Chains & Templates

### 8.1 User Chains

Users may create a chain:

```
Compressor → EQ → Saturator → Limiter
```

Chains are saved as:
```
PluginChain {
  chainId;
  name;
  nodes[];  // plugin nodes with parameters
}
```

Chains may also include:
- NodeGraph connections,
- routing specifics.

### 8.2 Factory Chains

Loophole provides:
- mixing chains,
- mastering chains,
- sound design chains.

### 8.3 Composer-Suggested Chains

Composer suggests:
- commonly used plugin combinations,
- orderings that match style/genre,
- per-project adaptive chains.

---

# 9. Handling Duplicates & Conflicts

## 9.1 Hiding Duplicates

Users may hide:
- old AU versions,
- redundant VST3 variants,
- CLAP versions when VST3 preferred (or vice versa).

Hidden plugins:
- still load in old projects,
- but do not appear in browser lists.

## 9.2 Missing Plugins

When loading a project:
- Pulse detects missing plugin variants,
- tries alternate variants automatically,
- if unavailable, Composer provides suggestions.

User may:
- choose a replacement plugin,
- disable the node,
- freeze the track,
- import plugin state from external file.

---

# 10. Plugin Presets

## 10.1 Preset Scopes

- Factory presets (from plugin vendor)
- User presets (stored per pluginId)
- Project presets (stored inside project)
- Chain presets (multi-plugin)

Preset metadata:

```
PluginPreset {
  presetId;
  pluginId;
  name;
  parameters[];
  rawState;
  tags[];
}
```

## 10.2 Cross-Format Preset Migration

If the user switches VST3 ↔ CLAP:
- Pulse maps parameters based on parameter aliasing,
- Composer provides statistical mapping,
- User can fix mismatches via UI.

---

# 11. Library Persistence

Pulse stores:

- user overrides (names, categories, tags),
- hidden plugins,
- presets,
- chains,
- Composer refinements,
- browsing state (recent plugins, favourites).

All stored in:

```
LibraryState {
  version;
  userOverrides[];
  favourites[];
  hidden[];
  chains[];
  categories[];
  tagAssignments[];
}
```

Stored per machine + optional cloud sync in future versions.

---

# 12. Interaction with Composer

Composer improves plugin library quality by:

- learning which plugins users tag similarly,
- learning how users categorise plugins,
- crowd-sourced deduplication rules,
- parameter alias mappings,
- plug-in replacement suggestions,
- heuristics for deterministic/non-deterministic classification.

Pulse uses Composer hints as advisory input.

---

# 13. IPC Responsibilities

### Aura → Pulse
- `pluginLibrary.open`
- `pluginLibrary.search`
- `pluginLibrary.rename`
- `pluginLibrary.tag.add/remove`
- `pluginLibrary.category.assign`
- `pluginLibrary.variant.select`
- `pluginLibrary.hide`
- `pluginLibrary.favourite`
- `pluginChain.create/edit/delete`
- `pluginPreset.save/load`
- `plugin.replace`
- `plugin.insertAt`

### Pulse → Aura
- `pluginLibrary.updated`
- `pluginLibrary.searchResults`
- `pluginLibrary.suggestion`
- `plugin.missing`
- `plugin.conflict`

### Pulse → Signal
- (Handled under plugin lifecycle)
- `plugin.instance.create`
- `plugin.instance.destroy`
- `plugin.state.apply`

---

# 14. UX Expectations

The Plugin Browser must:

- be extremely fast,
- feel like a modern search engine,
- surface relevant plugins immediately,
- integrate with control surfaces,
- support keyboard-only workflows,
- display clear variant information,
- provide visual plugin health (crash history),
- integrate seamlessly with NodeGraph drop targets.

---

# 15. Future Extensions

- Plugin store integrations (optional)
- Neural metadata embedding for improved search
- Project-adaptive plugin sorting (“You often use X after Y”)
- Time-based plugin suggestions based on user activity
- Gesture insertion (e.g., pad controllers)

---

# 16. Summary

Loophole’s Plugin Library & Browser:

- unifies plugins across formats,
- removes duplicates and clutter,
- provides powerful search and tagging,
- integrates Composer intelligence,
- offers templates and chains,
- supports robust renaming and aliasing,
- ensures plugin identity stability,
- provides intuitive drag/drop workflows,
- and makes plugin browsing a creative, expressive part of music-making.

This architecture finally solves plugin organisation properly — something no
major DAW has ever executed well — and becomes a defining part of Loophole’s
user experience.
