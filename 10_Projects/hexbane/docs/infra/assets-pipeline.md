---
type: project
project: Hexbane
area: infra
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, infra, assets, races, environments, spell-visuals, ai-art]
sources: ["vault:10_Projects/hexbane/assety-i-pipeline.md", "client:Resources/Races/_tools/build_races.sh", "client:Resources/Races/_tools/blender/README.md", "client:Resources/Races/_tpose/README.md", "client:Resources/Races/*/PLACEHOLDER.md", "client:Game/ScenesV3/Components/RaceAnimationPreview.cs", "client:export_presets.cfg", "client:.claude/skills/race-maker/SKILL.md", "client:.claude/skills/env-concept/SKILL.md", "client:.claude/skills/env-maker/SKILL.md", "client:Resources/Environments/*/environment.json", "client:Resources/SpellVisuals/README.md", "client:Resources/Images/Arenas/PROMPTS.md"]
---

# Asset pipeline: how art is produced and where it lands

Everything below is the client repo (`client:` = `~/RiderProjects/hexbane`). All numbers were taken from the working tree on 2026-09-07 (`ls`, PNG headers, grep). Almost none of the race art is committed yet: under `Resources/Races/` only the two `_tpose` files are tracked; the six race folders, `3d/` and the new `_tools/` scripts are untracked or staged (see [[repos-and-branches]]).

## Races (`Resources/Races/<race_id>/`)

### What exists now

| Race | Stills | Animation clips in `frames.tres` / `frames_sd.tres` | Extra |
|---|---|---|---|
| `human` | `avatar.png` only (real ChatGPT portrait); **no `front.png` / `idle.png`** | 16 (see list) | `depth/`, `hand_tracks.{json,tres}`, `meditation_tracks.tres`, `PLACEHOLDER.md` |
| `elf`, `dark_elf`, `shadow`, `gnome`, `orc` | `avatar.png` (real), `front.png`, `idle.png` (placeholders copied from the archived race *caelhart* on 2026-09-02, per each `PLACEHOLDER.md`) | 16 | same |

Clips (identical set for all six, `grep '"name"' Resources/Races/human/animation/frames.tres`): `idle, spell_throw, cast_2h, attack_1h_01..03, attack_2h_01..05, area_2h_01..02, meditation_enter, meditation, meditation_exit`.

Each clip folder `animation/<clip>/` holds `<clip>_sheet.png` (HD), `<clip>_sheet_sd.png` (SD), and a `render/` folder of raw frames. `render/` is gitignored (`client:.gitignore:5-6`) and `.gdignore`d, worth ~2 GB (`blender/README.md:117-118`); it is why each race directory is ~390–420 MB on disk while the tracked payload is the sheets + `.tres`.

Sheet geometry (human, same for all): 8 columns; HD sheets are 3640 px wide (frame height 512), SD sheets 2280 px wide (frame height 320); heights vary by frame count (idle HD 3640×2048, meditation HD 3640×4096, meditation_enter HD 3640×1024). Frame rate 15 fps (`build_races.sh:60-64`).

Support folders:
- `Resources/Races/3d/<race>/` — rigged Tripo FBX + `.fbm` texture folder + `meditation/<race>_meditation.blend` (112 MB total). `Resources/Races/3d/Magic Spell Pack/` — the 13 shared Mixamo clip FBXs.
- `Resources/Races/_tpose/<race>_tpose.png` — T-pose reference sheets generated 2026-09-03 in ChatGPT from `Resources/Images/CharCreate/Portraits/<race>.png`, used as Tripo image-to-3D input; not loaded at runtime (`_tpose/README.md`).
- `Resources/Images/CharCreate/Portraits/<race>.png` — the six portraits (same files as `avatar.png`).

### Current pipeline: Tripo → Mixamo → Blender → sprite sheets (`Resources/Races/_tools/`)

`build_races.sh [race_id ...]` (zsh, `client:Resources/Races/_tools/build_races.sh`) runs everything for one or all races, ~4 min per race:

1. Inputs: rigged character `~/3D/<source folder>/Magic Spell Pack/tripo_convert_*.fbx` (+ `.fbm`), copied into `Resources/Races/3d/<race>/` (`:18` RACES map: `human elf "dark elf" shadow gnome orc`; `:13-15` `PACK`, `MODELS=~/3D`, `BLEND_OUT=~/hexbane-archive/Models-2026-09-03/blend`). A race that ships its own clip pack (`standing idle.fbx` present) uses it instead of the shared pack (`:36-38`).
2. `blender/assemble_mixamo.py` → one `.blend` with the 13 clips mapped to game names (`:21-22` ANIMS/CLIPS, e.g. `spell_throw` = "standing 1H cast spell 01", `attack_1h_01` = "Standing 1H Magic Attack 01").
3. `blender/render_sprites.py` → PNG frames, `--fps 15 --size 1024 --angle 20` (`:56`).
4. `build_frames_from_renders.py <race> --fps 15 --height 512 --cols 8` → HD sheets + `frames.tres`; same with `--height 320 --variant sd` → `*_sheet_sd.png` + `frames_sd.tres` (`:60-64`).
5. Meditation: `blender/create_meditation.py --render` (13-frame enter, 60-frame loop, 13-frame exit), `pack_meditation.py` appends them to both variants, `blender/export_meditation_tracks.py` writes `meditation_tracks.tres` (`:67-72`; `README.md:126-140`). `export_hand_tracks.py` produces the palm tracks used to anchor cast VFX.

Manual steps, editing a clip in Blender, and the Tripo-rig legacy (`setup_edit.py`) are in `client:Resources/Races/_tools/blender/README.md` (steps 1–5, "Legacy: Tripo rig"). Check a race in the editor with `Game/ScenesV3/Dev/RaceAnimTest.tscn` (F6).

Adding a race: drop the Tripo download into `~/3D/<race>/`, add it to the `RACES` map, run the script, add the server row in `server:db/migrations/000002_reference_data.up.sql` and a portrait in `Images/CharCreate/Portraits/` (`README.md:17-20`; see [[progression]] "Races").

### Legacy pipeline: ChatGPT chroma sheets (`race-maker` skill)

`client:.claude/skills/race-maker/SKILL.md` still describes the 2026-06 route: a fully autonomous skill that invents a race, writes `character_bible.md`, and drives the ChatGPT desktop app (web fallback) to generate three stills (Idle/Avatar/Front) and six 3×3 sheets (`idle, spell_throw, meditation, victory, dead, damage_taken`) on a solid chroma background (green by default; magenta/blue when the palette collides), no painted glow, ~30 px frame margin. Companion tools: `_tools/key_sheet.py` (chroma key), `_tools/build_race_frames.py` (old sheet → `frames.tres`), `.claude/skills/race-maker/scripts/{extract_grid.py, slice_sheet_alpha.py, upscale_frames.jsx}` (untracked). **None of the six shipped races were produced this way**; the 35-race roster it made was archived to `~/hexbane-archive/Races-2026-08-31/` (never committed).

### HD vs SD selection at runtime

- Loader: `RaceAnimationPreview.GetRaceFramesPath(raceId)` (`client:Game/ScenesV3/Components/RaceAnimationPreview.cs:37-46`) picks `frames_sd.tres` or `frames.tres` by `PreferredVariant()` (`:56-64`): project setting `hexbane/graphics/race_sprites` = `hd` | `sd` | `auto` (default, `client:project.godot:69`); `auto` = SD on `mobile`/`web` features, HD elsewhere. Falls back to the other variant if the preferred file is missing (`:41-44`).
- Path pattern: `res://Resources/Races/<race>/animation/frames[_sd].tres` (`:48-53`).
- Export presets drop the unused set: Windows/macOS exclude `*_sheet_sd.png` + `frames_sd.tres` (`client:export_presets.cfg:11,308`); Android/iOS exclude `*_sheet.png` + `frames.tres` (`:82,566`).
- In-match use: `RaceSpriteAnimator` (`client:Game/ScenesV3/Components/RaceSpriteAnimator.cs:19`) loads the same resource, falls back to `human`, loops `idle`, and picks the cast clip via `CastAnimationResolver` (`spell.animation` from server → `attack_1h_01` → `spell_throw`), time-scaled to the cast (1×–6×) (`blender/README.md:185-210`). Both duelists render their own race from `race_id` in the player views; the bot gets a random race (`server:modules/match/ai_match/init.go`, `randomBotRace`) — see [[combat-v2]].

## Environments / arenas

- `Resources/Environments/<env_id>/` — three parallax sets: `gilded_waste`, `planet_of_wind`, `prism_reach`. Each has `environment_bible.md`, `environment.json`, `draft/`, and layers `sky, far, mid, platform, foreground` (+ `fx.png` only for `prism_reach`). `environment.json` declares `design_resolution [2400,1080]`, `layer_resolution [2400,1600]`, per-layer `z`, `scroll` factor (0.10 → 1.15) and, for `fx`, `blend: add` + `animate: drift+pulse`.
  Produced by the `env-concept` skill (bible + master concept prompt for ChatGPT) and `env-maker` skill (per-layer prompts, ChatGPT upscale, Adobe background removal, manifest) — `client:.claude/skills/{env-concept,env-maker}/SKILL.md`.
  **Runtime use:** referenced only from dev scenes `Game/ScenesV3/Dev/ArenaMaps/{ArenaMapsDev,VerifyArenaMaps}.cs` (grep `Resources/Environments` in `Game/`). Not used by the live duel.
- `Resources/Images/Arenas/{emerald,glacier,forge}.png` (1672×941) — the arenas the live duel actually shows, listed in `client:Game/ScenesV3/ReferenceDuel/ArenaCatalog.cs:14-16` (Polish display names "Szmaragdowe Sanktuarium", "Lodowa Katedra", "Obsydianowa Kuźnia"). Generated September 2026 with an image model using `Resources/Images/ReferenceDuel/arena.png` as composition reference; atmosphere/lighting is done in Godot shaders (`LivingSky.gdshader`, `BiomeAtmosphere.gdshader`) (`Images/Arenas/PROMPTS.md`).
- `Resources/Images/ReferenceDuel/{arena,duelists,frames,hud,ornaments,sky_mask}.png` — cut-outs for the reference duel HUD (`PROMPTS.md` alongside).
- `Resources/Backgrounds/{Floors,Sky,Walls}` — old tile/parallax backgrounds, referenced only by the orphaned `GameHud/Main.tscn` and `Dev/HudPreview.cs`.

## Spell visuals

- `Resources/SpellVisuals/<spell_id>.tres` — per-spell cast-presentation presets (`fireball, magic_arrow, magic_reflection, mirror_reflection`, `_template.tres`), authored in the dev scene `Game/ScenesV3/Dev/ArenaMaps/ArenaMapsDev.tscn` and read by `RaceSpriteAnimator.BeginCast` through `SpellVisualPreset.cs`. A server-sent `visual_key` (`vfx1_…`) overrides the local preset; format in [[spell-visual-key]]. (`Resources/SpellVisuals/README.md`, Polish.)
- `Application/Modules/Spell/Effects/SpellEffectConfigurations.cs:14-40` — the older projectile/static effect registry: 10 active ids (`magic_sparkle, heal, restore, cure, poison_dart, flamestrike, gust, stoneguard, mirror_ward, spark`), 10 commented out. **None of the 14 catalog ids** (see [[spell-system]]) is configured there; scenes live in `Game/FX/*.tscn` (21).
- `Resources/Spells/<id>/` — 21 folders, mostly the pre-redesign prototype: 10 carry the full n8n bundle (`spell.json`, `manifest.generated.json`, `primitives/ textures/ motions/ references/`), the rest just `<id>.png`. Only `magic_arrow` (icon) and `mirror_reflection` (`sfx/`) match the current catalog. `Resources/Icons/` = 31 legacy SVG icons. `Resources/Effects/{paralyze,poison,reflection,shield}.png` = status icons. `Resources/Textures/Spells/shield`.
- Old generator inputs: `prompt.md`, `vfx.md`, `spell_output.md` (repo root, n8n prompts; `vfx.md` called the server RPC `get_spell_details_yaml`) and `.claude/commands/create_spell_{icon,vfx,textures,sfx,sounds}.md`. Whether the n8n workflow still exists: unverified.

## UI images and references

- `Resources/Images/<Screen>/` — per-screen cut-outs: `Auth` (10 files), `CharCreate` (33 incl. `Icons/`, `Portraits/`), `Detail` (30), `Menu` (20), `Lobby` (14), `GameHud` (20), `GameStats` (9), `Frames` (25), `Bar` (15), `Backgrounds` (1), `ReferenceDuel` (7). Produced 2026-08-31..09-04 by handing ChatGPT crops of the reference mocks and chroma-keying (`key_green.py`; see memory notes), never by prose prompts.
- `Resources/Images/UI/` — four third-party asset packs (`Fantasy UI Borders`, `UI Adventure Pack`, `UI Pack`, `UI Pack - Adventure`; ~2100 files).
- `reference/<screen>/*.png` — the design mocks every ScenesV3 screen was rebuilt from: `auth screen/full.png`, `char create/{race,rename,spells,stats,summary}.png`, `menu/{main_screeen,news,settings,social,spellbook,stats,summary}.png`, `lobby/{lobby,rearange}.png`, `fight/{game,old,before-reference-variant}.png`, `game_stats/{victory,defeat,old_one}.png`. Tracked partially (26 status entries). `reference/fight/game.png` is oddly a csproj `<Content>` item (`client:hexbane.csproj:33`).
- Fonts: `Resources/Fonts/{Cinzel, Cormorant_Garamond, Inter, Marcellus}` (79 untracked files; 48 old font files deleted in the working tree).
- Audio: `Resources/Music/menu/{battle.mp3, lobby.wav, game_found.wav, afterlobby.wav, "Whisper of the Arcane" stems}`, `Resources/Music/heal.wav`, `Resources/Sound/{fireball.wav, heal.mp3}`. Spell SFX otherwise absent.
- Utilities: `Scripts/remove_bg.py` (rembg U2Net, run with `uv`), `remove_bg.sh`; Godot MCP addon `addons/godot_mcp` + `mcp_server/` used for in-editor verification.

## Archives outside the repo (this Mac)

- `~/hexbane-archive/Races-2026-08-31/` — the withdrawn 35-race ChatGPT roster (vault says 922 MB incl. `_raw_chroma/`, `_style_comparison/`, `_server-docs-client-races/`; contents not re-verified).
- `~/hexbane-archive/Models-2026-09-03/blend/` — assembled `<race>_mixamo.blend` files written by `build_races.sh`.
- `~/3D/{human, elf, "dark elf", shadow, gnome, orc}/` — Tripo/Mixamo downloads, the pipeline's source models.

## Source of truth in code
- `client:Resources/Races/_tools/build_races.sh`, `_tools/blender/*.py`, `_tools/build_frames_from_renders.py`, `_tools/pack_meditation.py` — the race build pipeline, clip list, resolutions
- `client:Resources/Races/_tools/blender/README.md` — step-by-step procedure, meditation clips, in-game clip selection
- `client:Game/ScenesV3/Components/RaceAnimationPreview.cs` — HD/SD path selection and `hexbane/graphics/race_sprites`
- `client:Game/ScenesV3/Components/{RaceSpriteAnimator,CastAnimationResolver,SpellVisualPreset}.cs` — in-match sprite, clip choice, visual presets
- `client:export_presets.cfg` — per-platform sprite-set exclusion
- `client:Game/ScenesV3/ReferenceDuel/ArenaCatalog.cs` — arenas the live duel uses
- `client:Resources/Environments/*/environment.json` — parallax layer manifests
- `client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs`, `client:Game/FX/*.tscn` — legacy projectile/static VFX registry
- `client:.claude/skills/{race-maker,env-concept,env-maker}/SKILL.md` — generation procedures (race-maker = legacy route)
