---
type: project
project: Hexbane
area: infra
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, infra, assets, races, art-pipeline]
---

# Asset locations and production pipeline

## Runtime tree

```text
Resources/
  Arenas/
    Storm/background.png
    Emerald/background.png
    Glacier/background.png
    Forge/background.png
    Shared/sky_mask.png
  Audio/
    Music/
      Menu/whisper_of_arcane.wav
      Lobby/draft.wav
      Lobby/arrangement.wav
      Battle/battle.mp3
      Results/{victory,defeat}.mp3
    UI/game_found.wav
    Spells/<spell-or-cast-family>/<stage>.wav
  Branding/
    logo_fire.png
    AppIcon/                      # launcher/adaptive icons
  Fonts/<family>/                  # font files and original license notices
  Races/<race_id>/
    avatar.png                    # single copy, including character creation
    front.png, idle.png, ...       # available race presentation images
    animation/
      frames.tres, frames_sd.tres
      hand_tracks.json, hand_tracks.tres, meditation_tracks.tres
      depth/<clip>.png
      <clip>/<clip>_sheet.png
      <clip>/<clip>_sheet_sd.png
  Spells/
    Icons/<canonical_id>.png       # all fourteen spells
    CastPresets/<canonical_id>.tres
    CastPresets/_template.tres
  UI/
    Auth/
    CharacterCreation/Icons/
    Dashboard/
    Lobby/
    Loading/
    Combat/                       # current ornaments and frame atlas
    Tutorial/swipe_hand.svg
    Shared/{Backgrounds,Icons,Shaders}/
```

Use canonical spell names to locate icons: `magic_arrow`, `mirror_reflection`, `firebolt`, `heavy_bolt`, `delayed_hex`, `poison`, `paralysis`, `cleanse`, `mend`, `greater_heal`, `regeneration`, `barrier`, `dispel`, `consume_venom`. `Spell.GetIconPath` accepts legacy ids (e.g. `fireball`, `arcane_shield`) and legacy full resource paths, then resolves the canonical icon. Retired color avatar values resolve to null, allowing each screen to show the existing race portrait; valid `res://` and `user://` custom/race paths remain supported. Race paths and HD/SD selection are unchanged.

`Shared/Icons` contains stat, skill, modifier, settings, news and small UI symbols used across screens; filename prefixes identify their role. The legacy HUD textures, scenes and exclusive components were subsequently deleted at the user's explicit request. Shared procedural helpers (`HudArt`, `HudPanel`, `EffectSlot`) remain because the current duel uses them; they no longer load legacy art.

## Editable inputs and tools

```text
ArtSource/                        # .gdignore: never imported or exported by Godot
  Races/
    Models/<race>/                # FBX, textures and editable meditation/death blends
    Models/Magic Spell Pack/      # shared Mixamo clips
    TPose/                        # source reference sheets
    Tools/
      build_races.sh
      build_frames_from_renders.py
      pack_meditation.py, pack_death.py, export_hand_tracks.py
      blender/*.py, raster_depth.c
  UI/
    Tools/key_green.py
  Audio/Music/                    # source stem archives
```

Art-source files are retained because they support editing and regeneration, even though the game does not load them. Do not treat absence of a runtime reference as proof an authoring input is unused. Original font license/readme notices remain with fonts.

Run the race pipeline from the repository root:

```sh
ArtSource/Races/Tools/build_races.sh human
```

The default source-model input remains `~/3D` (`MODELS` override); clips default to `ArtSource/Races/Models/Magic Spell Pack` (`PACK` override). `BLENDER` and `BLEND_OUT` retain their existing overrides. Script-relative project roots were updated and Python syntax/zsh syntax checked; a full Blender rerender was not run in this migration.

Intermediate Blender frames are recreated in `Resources/Races/<race>/animation/<clip>/render/`, gitignored and `.gdignore`d by the renderer. They were removed during cleanup. Rebuild renders before running packers/track regeneration; packed HD/SD sheets, depth atlases, palm tracks and meditation tracks remain ready for game use. Export filters selecting HD for desktop and SD for mobile remain unchanged.

Detailed original authoring notes moved into this vault under `docs/infra/asset-sources/`; see [Blender workflow](asset-sources/Races/_tools/blender/README.md), [T-pose sources](asset-sources/Races/_tpose/README.md), [arena prompts](asset-sources/Images/Arenas/PROMPTS.md), [combat artwork prompts](asset-sources/Images/ReferenceDuel/PROMPTS.md). These contain historical inventory/steps; this page defines current locations. Spell icon style: [[spell-icon-art-direction]]. Spell audio and prompts: [[spell-audio]] plus the JSON manifest beside the audio.

## Cleanup and validation — 2026-09-15

Migration inventory: [file manifest](../audits/2026-09-15-resource-migration.json), including original paths, destinations, hashes and deletion reasons. Removed unused loose images, six byte-identical portrait copies, old mirror sounds superseded by ElevenLabs streams, the unreferenced `magic_reflection` preset, waveform/OS/Python/Blender backup caches and rebuildable raw render frames. Removed unreferenced `GameHud/Main.tscn` and `GameHud/ArcaneDuel/Main.tscn` and their exclusive assets after detecting broken legacy tile atlases. Shared player/HUD components and their preview assets remain.

Preserved all 196 moved binary files byte-for-byte and all 155 tracked import UIDs. All 77 compiled C# canonical/alias/persisted-path/file-existence checks passed. Opened all twelve relocated Blender files in Blender 5.2 and verified zero missing external image dependencies. Godot filesystem rescanned in the running editor and headless import completed. All 639 resources/scenes in the load audit returned non-null; no resource-load or atlas errors remain. The generic audit logs two ObjectDB instances/one resource still in use at shutdown; this is a cleanup warning from the audit process, not evidence of a missing asset. The headless editor also logs an EditorSettings shutdown warning. C# build: zero errors, eleven existing warnings. GPU-rendered GameplaySandboxVerification passed all fourteen icons/casts, reset cancellation, reverse reflection/dodge, meditation/audio and held status; screenshot visually reviewed. `git diff --check` passed. The missing tutorial swipe image found in the baseline was restored as SVG and loads in Godot.

Evidence files: client `verification/resource-organization/` (summary, alias-check output, Blender audit, load-audit script and gameplay screenshot).

No fresh platform export, physical-device run, live Nakama match or full Blender regeneration was performed. The pre-existing `export_presets.cfg` edit was preserved byte-for-byte. No commit or push.

## Source of truth in code

- client:Resources/
- client:ArtSource/Races/Tools/build_races.sh
- client:ArtSource/Races/Tools/blender/*.py
- client:Core/Spells/Spell.cs
- client:Core/Characters/Character.cs
- client:Game/ScenesV3/Components/{RaceAnimationPreview,SpellVisualPreset}.cs
- client:Game/ScenesV3/ReferenceDuel/{ArenaCatalog,ReferenceHud,LivingArena}.cs
- client:Game/Autoloads/MenuPlayer.cs
- client:Scripts/Audio/{generate_spell_audio,generate_result_music}.py
- client:export_presets.cfg

## Follow-up: remove remaining legacy resources (2026-09-15)

Explicit user request: remove legacy resources. Deleted 112 additional files (45.09 MiB): all UI/LegacyHud textures/imports, Arenas/Legacy sky, UI/Avatars color portraits, obsolete PSD bar sources, old HUD implementation and its two previews, unused avatar card and old bar/spell/effect components. Inventory: [legacy removal manifest](../audits/2026-09-15-legacy-resource-removal.json).

DuelV2Preview now always tests ReferenceDuel/MainReference. Shared current-HUD helpers and live actor/controller code remain. Status icons resolve through the maintained spell icon catalog instead of the absent Resources/Effects directory. Dashboard, lobby and character creation resolve race portraits from Resources/Races/<race>/avatar.png. Retired avatar values fall through to each screen's race fallback.

Build: 0 errors/11 existing warnings. DuelV2Preview passed: 9 slots, layout, all 14 icons, enemy cast label, effect apply/remove, queue/ticks and reconnect retry. The offline harness logs its unauthenticated notification initialization issue and shutdown resource warnings; these are not missing-resource failures. Previous migration counts/compatibility tests above describe the earlier state before this explicit legacy deletion. No device/export/live-match claim.

- Final follow-up verification: all 598 retained resources/scenes loaded; no remaining references to deleted files, no broken scene/resource dependencies and no Legacy directories under Resources. Rendered GameplaySandboxVerification passed all fourteen casts/icons plus reflection/dodge, cancellation and meditation/audio; screenshot reviewed. Godot editor filesystem refreshed; git diff --check passed.
