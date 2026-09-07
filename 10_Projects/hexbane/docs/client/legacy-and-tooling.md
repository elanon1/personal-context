---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, tooling, legacy, gotchas]
sources: ["vault:10_Projects/hexbane/architektura-klienta.md (2026-08-31)", "vault:10_Projects/hexbane/assety-i-pipeline.md (2026-08-31)", "vault:10_Projects/hexbane/protokol-klient-serwer.md (2026-08-31)"]
---

# Legacy, tooling and known dead spots (client)

Facts preserved from the 2026-08-31 reconnaissance that no other note covers. Verified on
2026-09-07 unless marked *(unverified)*.

## Dead or legacy code

- `SceneManager` still declares route keys `startup`, `autoload`, `debug_rpcs` pointing at
  `Game/Scenes/…`, a folder that no longer exists (`client:Game/Autoloads/SceneManager.cs:32`).
- `GodotGTweensContextNode` is an autoload with **zero users** — every animation uses the native
  `Tween`. Safe to remove together with `Plugins/GTweens/`.
- `Game/ScenesV3/GameHud/Main.tscn` and `GameHud/ArcaneDuel/Main.tscn` are orphans: the live match
  scene is `ReferenceDuel/MainReference.tscn` (see [[duel-v2-client]]); only `Dev/HudPreview.cs`
  references the old HUD.
- `Core/Common/Enums/Opcodes.cs` still declares the retired combat opcodes 11–15 and 21–28; the match
  dispatcher excludes them (see [[opcodes]]).
- `Core/Spells/Spell.cs` maps `mirror_reflection → mirror_ward`, `firebolt → fireball`,
  `heavy_bolt → flamestrike` as legacy asset aliases; `Core/Spells/StandardSpells.cs` cites a YAML path
  that no longer exists on the server.
- `Core/Characters/Character.cs` hard-codes `MaxSpellSlots = 6` until the server value arrives.
- The client calls RPCs the server does not register: `start_story` (commented out on the server) and
  `get_users` (never existed). See [[rpcs]].
- Socket `Closed` → reset + logout; there is **no automatic socket reconnect** outside the duel
  rejoin path described in [[duel-v2-client]]. *(GameContext 30 s session/socket timer: unverified)*

## Environment file

`.env` is read by the `EnvLoader` autoload in the editor only (dotfiles are never packed into
exports — see [[deploy-android]]). Keys used in code: `NAKAMA_SERVER` (`local|synology|prod`),
`NAKAMA_HOST`, `NAKAMA_PORT`, `DEV_AUTO_LOGIN_MODE` (`existing|new_character`), `AUTH_EMAIL`,
`AUTH_PASSWORD`; `DEV_AUTO_LOGIN` and `DEBUG_SCENE` sit in `.env` but are not read through
`EnvLoader.Get` *(unverified how)*. There is no `.env.example`. The file is git-tracked and packed as
csproj `Content` — it holds dev credentials (open question in [[10_Projects/hexbane/_state|_state]]).

Server presets are hard-coded in `Application/Nakama/NakamaClientManager.cs`: `local`, `synology`
(:7350), `prod` (`hexbane.elanon.pl:443`).

## n8n spell pipeline (legacy)

Root files `prompt.md` (spell description → JSON with icon and audio prompts), `vfx.md` (definition
→ `vfxBlueprint`; calls the server RPC `get_spell_details_yaml`) and `spell_output.md` (schema of
`spell.json`) are prompts for an n8n workflow that produced the 10 old-prototype folders in
`Resources/Spells/<id>/` (`spell.json`, `manifest.generated.json`, `primitives/textures/motions/references`,
Google Drive links). Whether the workflow still runs is unknown; none of the 14 current spell ids went
through it. `spell_output.md` is still listed as csproj `Content`. Related command: `old_create_spell` (the five `.claude/commands/create_spell_*` commands were removed on 2026-09-07). Formerly also:
`.claude/commands/create_spell_*` and `old_create_spell`.

## Race art history

- `~/hexbane-archive/Races-2026-08-31/` (922 MB, never in git): the withdrawn 35-race ChatGPT roster,
  `_raw_chroma/`, `_style_comparison/`, and the old race bibles.
- The `race-maker` skill (`.claude/skills/race-maker/`) describes the ChatGPT chroma-sheet method
  (solid chroma per race, green `#00FF00` default, no painted glow, ~60–65 % of the cell, ~30 px
  margin, frame 1 = idle, one ChatGPT thread per race). Its helper scripts `extract_grid.py`,
  `slice_sheet_alpha.py`, `upscale_frames.jsx` are untracked in git. The shipped six races were built
  with the Blender/Mixamo pipeline instead — see [[assets-pipeline]].
- `Resources/Icons/` holds 31 legacy SVG icons; spell audio is nearly absent
  (`Resources/Sound/{fireball.wav,heal.mp3}` plus the ElevenLabs set for `mirror_reflection`).

## Editor tooling

- Addons: `godot_mcp` (Godot MCP Pro, WebSocket :6505, Node server in `mcp_server/`; version
  *(unverified)*), `ColorPreview`, `rider-plugin`, `hexbane_android` (**must stay enabled**, declares the
  `hexbane://` scheme), `GodotPlayGameServices` (**disabled**, see [[social-sign-in]]).
- `.claude/`: 7 agents, 29 commands, 3 skills (`race-maker`, `env-concept`, `env-maker`);
  `MAX_THINKING_TOKENS=32000` in `.claude/settings.json`.
- Godot MCP gotchas (stale `get_game_screenshot`, dead `click_button_by_text` on ScenesV3 buttons, use
  `execute_game_script` + `force_draw`/`save_png` and `emit_signal("pressed")`) live in Claude Code's
  project memory, not in the repo.
- `Scripts/remove_bg.py` (rembg via `uv`) removes backgrounds for generated art.

## Source of truth in code
- `client:Game/Autoloads/SceneManager.cs` — route table incl. dead keys
- `client:project.godot` — autoload list, `[hexbane]` settings
- `client:Application/Nakama/NakamaClientManager.cs` — server presets and host resolution
- `client:Core/Spells/Spell.cs`, `client:Core/Spells/StandardSpells.cs` — legacy id aliases
