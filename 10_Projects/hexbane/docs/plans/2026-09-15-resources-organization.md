---
type: project
project: Hexbane
area: infra
status: complete
created: 2026-09-15
updated: 2026-09-15
verified: 2026-09-15
---
# Resources organization implementation plan

**Goal:** Make resource locations discoverable by purpose and remove proven unused files while preserving Godot references and art production inputs.
**Architecture:** Runtime assets stay in Resources, organized into UI, Arenas, Branding, Audio, Fonts, Races and Spells. ArtSource holds editable source material and tools behind .gdignore. Preserve imported resource UIDs; update source paths and reimport in Godot.
**Tech stack:** Godot 4.7 .NET, C#, Python/Blender pipeline.
**Spec:** User request on 2026-09-15: organize Resources, repair Godot, delete unused assets.

## Constraints
- Preserve the pre-existing export_presets.cfg edit. No commit or push.
- Dynamic spell, race, UI and audio paths count as usage. Keep authoring source inputs and font licenses.
- Documentation belongs in this vault only.

## Tasks
- [x] Inventory static and dynamic consumers; record baseline missing paths and moved/deleted file hashes.
- [x] Move UI by screen/shared role, arena art and branding to named directories; group music with audio and spell presets/icons together. Consolidate identical race portraits.
- [x] Move models, tools, T-pose sources, PSDs and music sources to ArtSource; remove unused artwork, old ward sounds, waveform caches, rebuildable raw sprite frames and obsolete preset. Move existing in-repo art notes to vault.
- [x] Update scene/C#/shader/import/config/tool paths, generator-relative roots and editor favorites; preserve UIDs and export filters.
- [x] Build C#, import in Godot, load all retained resource dependencies and exercise offline presentation. Compare against baseline; record limitations.
- [x] Update assets-pipeline, affected contracts, index, decision log and session log.

## Source of truth in code
- client:Resources/
- client:Core/Spells/Spell.cs
- client:Game/ScenesV3/Components/SpellVisualPreset.cs
- client:ArtSource/Races/Tools/
