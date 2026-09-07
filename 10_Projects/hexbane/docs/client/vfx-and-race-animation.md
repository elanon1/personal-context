---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, races, animation, vfx, arena]
sources: ["client:Resources/Races/_tools/blender/README.md", "client:docs/client/arena-maps/README.md", "client:docs/client/cast-charge/README.md", "client:docs/client/gesture-occlusion/README.md", "client:docs/client/gesture-vfx/README.md", "client:docs/client/meditation/README.md", "client:docs/client/mirror-reflection/README.md", "client:docs/client/reference-duel/README.md", "client:Resources/SpellVisuals/README.md", "client:CLAUDE.md"]
---

# Race sprites, gesture VFX and the living arena

Six races: `human, elf, dark_elf, shadow, gnome, orc` (`Resources/Races/<race>/`, `Core/Characters/RaceCatalog.cs`). Spell FX and icons are covered in [[spell-vfx-configuration]]; the HUD in [[duel-v2-client]].

## Sprite pipeline (`Resources/Races/_tools/`)

`build_races.sh [race …]` runs the whole chain per race: Mixamo-rigged Tripo FBX + "Magic Spell Pack" clips → `blender/assemble_mixamo.py` (.blend) → `blender/render_sprites.py` (PNG frames, 20° camera, 15 fps) → `build_frames_from_renders.py` (sheets + `frames.tres`), twice: `--height 512` (HD) and `--height 320 --variant sd` (`frames_sd.tres`, `*_sheet_sd.png`). Blender at `/Applications/Blender.app`, models under `~/3D`, archived blends in `~/hexbane-archive/Models-2026-09-03/blend`. The raw render folders and `Resources/Races/3d/` carry `.gdignore`. Full step-by-step: `client:Resources/Races/_tools/blender/README.md`.

Clips per race (identical for all six, `Resources/Races/<race>/animation/*/`, names inside `frames.tres`):

| Group | Clips |
|---|---|
| 13 Mixamo clips | `idle, spell_throw, cast_2h, attack_1h_01..03, attack_2h_01..05, area_2h_01, area_2h_02` |
| Meditation (authored in Blender, `create_meditation.py` + `pack_meditation.py`) | `meditation_enter` (13 f), `meditation` (60 f loop), `meditation_exit` (13 f) at 15 fps |

Per race also: `hand_tracks.tres` + `hand_tracks.json` (palm/chest/hips/head/elbow/foot projections and depth per frame, `export_hand_tracks.py`), `meditation_tracks.tres` (`blender/export_meditation_tracks.py`), `depth/<clip>.png` (13 depth atlases from `blender/depth_export.py` + `raster_depth.c`), `avatar.png`. `human/PLACEHOLDER.md` records that the human stills are placeholder art copied from the archived "caelhart" race (2026-09-02).

Runtime selection: `RaceAnimationPreview.GetRaceFramesPath` (`Game/ScenesV3/Components/RaceAnimationPreview.cs:37-64`) picks `frames_sd.tres` on `mobile`/`web` or when `hexbane/graphics/race_sprites=sd`, `frames.tres` otherwise (`hd` forces it); each export preset excludes the unused set ([[deploy-android]]). Coordinates in the track resources are normalized so HD and SD share them.

## `RaceSpriteAnimator` (`Game/ScenesV3/Components/RaceSpriteAnimator.cs`)

Node2D owned by `Player.tscn` and every dev scene. Constants: fallback race `human`, cast speed clamp 1×–6×, meditation enter 0.18 s / exit 0.16 s (lines 21-30). It owns three child effects: `CastCharge` (palm energy while casting), `GestureVfx` (per-clip shader effect) and `MeditationVfx`; plus `PoseOcclusion` (depth masking).

- **Cast**: `BeginCast` picks the clip via `CastAnimationResolver` (server `spell.animation` → `attack_1h_01` → `spell_throw`) and retimes the wind-up so its launch frame (dominant hand's max reach within the first 85 % of the clip) lands at the server-adjusted cast time; the pose holds until acceptance, then a 0.3 s recovery; failure cancels. Stale cast ids never cancel a newer cast. `ReconcileCast` resyncs to snapshot ticks.
- **Meditation**: `SetMeditating` plays enter → loop → exit, reverses mid-transition, yields to casts, resumes after recovery, falls back to idle when clips are missing. `Player._Process` feeds it the authoritative snapshot state for both duelists; paralysis, poison, death and match end stop it.
- **Gesture VFX**: 11 variants, one per `attack_*/cast_*/area_*` clip (`Impulse, Spiral, Discharge, Double weave, Energy crescent, Singularity, Embers, Gust, Seal, Ascent, Pressure wave`), 66 race×clip combinations; the shader clock derives from the current frame, so server-driven scrubbing, slow casts and mirroring stay in sync. Hand trails use the two previous poses. Effects clear on idle, interruption or race change.
- **Occlusion**: hand energy behind the body is hidden using the depth atlas and the sprite alpha; ground rings use a fixed support level from the idle pose and render under the sprite.
- **Mirror Reflection ward** (`Game/FX/MirrorWard.cs`): translucent shell around the silhouette; lifetime from the reflection status end tick (3 s in previews); consumption tears it into trails over 0.28 s away from the incoming spell; sounds `formation.wav` / `shatter.wav`.

Spell-specific overrides of the gesture (presets, `visual_key`) are described in [[spell-vfx-configuration]].

## Living arena (`Game/ScenesV3/ReferenceDuel/`)

`LivingArena.cs` replaces the static background in `MainReference.tscn` and `ReferencePreview.tscn`; `ArcaneMotes.cs` draws 18 motes on mobile, 28 on desktop (`ReferenceHud.cs:110`). Shaders: `LivingSky`, `BiomeAtmosphere`, `ChromaKey`, `CircleMask`, `RingKey`, `BarFill` (`.gdshader`, no GDScript). Inspector knobs on `ReferenceHud`: `SkyWindStrength`, `SkyAtmosphereStrength`, `ReducedAtmosphereMotion` (freezes clouds/fog, hides motes and blooms), `StormIllumination`.

Maps (`ArenaCatalog.cs:13-16`):

| Id | Name | Art |
|---|---|---|
| `storm` | Burzowa Cytadela | `Resources/Images/ReferenceDuel/arena.png` |
| `emerald` | Szmaragdowe Sanktuarium | `Resources/Images/Arenas/emerald.png` |
| `glacier` | Lodowa Katedra | `Resources/Images/Arenas/glacier.png` |
| `forge` | Obsydianowa Kuźnia | `Resources/Images/Arenas/forge.png` |

Selection: `ArenaMatch._EnterTree` (`ArenaMatch.cs:11-17`) calls `ArenaCatalog.ForMatch(matchId)` = SHA-256 of `"hexbane-arena-v1:" + matchId`, first byte mod 4 (`ArenaCatalog.cs:25`). Both peers and reconnects get the same map with no protocol message; do not reorder the catalog without versioning the prefix. Without a match id (F6) the map is chosen locally. All maps share the platform band (60–68 % of source height, foot baseline 62.5 %); atmosphere shaders never move source geometry. Prompts for the generated art: `Resources/Images/Arenas/PROMPTS.md`.

Mobile layout: players are positioned from the HUD safe-area transform with feet 28 design px above the highest spell button; the combat log sits between the spell groups (2 entries, 4 when expanded).

## Dev and verification scenes (`Game/ScenesV3/Dev/`, run with F6 or headless)

| Scene | Purpose |
|---|---|
| `ArenaMaps/ArenaMapsDev.tscn` | offline playground: map selector, **Losuj / Pauza / Ograniczony ruch**, per-side race + gesture + effect + tempo (0.25–1.5×), **Animacja + VFX gracza/przeciwnika**, loop toggle, **Bariera 3 s** / **Trafienie w barierę**, preset save/load (`Resources/SpellVisuals/<id>.tres`), **Kopiuj/Wczytaj klucz** (`visual_key`), **Rzuć zapisany czar**. Never sends match commands. |
| `RaceAnimTest.tscn` | pick race and clip |
| `MeditationTest.tscn` | headless: `Godot --headless --fixed-fps 60 --path . Game/ScenesV3/Dev/MeditationTest.tscn`, 12 race/resolution combos + Player snapshot wiring |
| `MeditationVfxPreview.tscn` | all six races; **M** meditate, **X** stop; `-- --capture` writes `docs/client/meditation/energy-preview.png` |
| `DuelV2Preview.tscn`, `GameHudPreview.tscn`, `HudPreview.tscn`, `StatsPreview.tscn`, `StarterSelectionCheck.tscn`, `TutorialVerification.tscn` | HUD / wizard / tutorial harnesses ([[duel-v2-client]], [[client-tutorial]]) |
| `ReferenceDuel/ReferencePreview.tscn` | offline HUD preview (exports `PreviewCapacity`, `PreviewUnlocked`, `PreviewMana`, races, portraits, safe-area insets) |

Verifier scenes write evidence into the repo (all under `.gdignore` except `reference-duel`):

| Verifier | Output dir (`res://docs/client/…`) | Writer |
|---|---|---|
| `ReferenceDuel/ReferenceVerification.tscn` | `reference-duel/` (`verification.json`, `state-checks.json`, `atmosphere-performance.json`, PNG/GIF) | `ReferenceDuel/VerifyPreview.cs:15` |
| `ArenaMaps/VerifyArenaMaps.tscn` | `arena-maps/` | `VerifyArenaMaps.cs:14` |
| `ArenaMaps/VerifyRaceCharge.tscn` | `cast-charge/` | `VerifyRaceCharge.cs:19` |
| `ArenaMaps/VerifyGestureVfx.tscn` | `gesture-vfx/` (`verification.json`, per race×clip PNGs, `layout-*.png`, `motion-*.png`) | `VerifyGestureVfx.cs:32,77,120,157` |
| `ArenaMaps/VerifyGestureOcclusion.tscn` | `gesture-occlusion/` | `VerifyGestureOcclusion.cs:22,84,127,132` |
| `ArenaMaps/VerifyMirrorWard.tscn` | `mirror-reflection/` (`burst-NN.png`, layouts) | `VerifyMirrorWard.cs:25,94` |
| `MeditationVfxPreview.tscn --capture` | `meditation/energy-preview.png` | `MeditationVfxPreview.cs:37` |
| `ArenaMaps/VerifyRaceGrounding`, `VerifyReflectionPreset`, `VerifySpellVisualKeys`, `VerifySpellVisualPresets`, `VerifyWardAudio` | console only | — |

Last recorded results (README claims, Godot 4.5.2 Compatibility on macOS, not re-run 2026-09-07): reference-duel 88/88, arena maps 44/44, gesture VFX 8898/0 failures, occlusion 12398/0, mirror ward 57/0, meditation 12/12 combos. None measured Android hardware performance or a live Nakama match. Headless runs also log the pre-existing missing `signal_lens` autoload and an unresolved theme UID. The generated PNG/GIF/JSON (≈ 800 MB across the seven folders) are evidence, not documentation.

## Source of truth in code
- `client:Resources/Races/_tools/build_races.sh`, `build_frames_from_renders.py`, `pack_meditation.py`, `export_hand_tracks.py`, `blender/*.py` — asset pipeline
- `client:Resources/Races/<race>/animation/` — `frames.tres`, `frames_sd.tres`, track and depth resources
- `client:Game/ScenesV3/Components/RaceAnimationPreview.cs` — HD/SD selection
- `client:Game/ScenesV3/Components/RaceSpriteAnimator.cs`, `CastCharge.cs`, `GestureVfx.cs`, `MeditationVfx.cs`, `PoseOcclusion.cs`, `CastAnimationResolver.cs`, `HandTrackData.cs` — runtime animation and effects
- `client:Game/FX/MirrorWard.cs`, `WardAudio.cs` — reflection ward
- `client:Game/ScenesV3/ReferenceDuel/ArenaCatalog.cs`, `ArenaMatch.cs`, `LivingArena.cs`, `ArcaneMotes.cs`, `*.gdshader` — arena
- `client:Game/ScenesV3/Dev/**`, `ReferenceDuel/VerifyPreview.cs` — dev scenes and verifiers
- `client:export_presets.cfg` — per-platform sprite set exclusion
