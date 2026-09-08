---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, client, tooling, legacy, gotchas]
sources: ["vault:10_Projects/hexbane/architektura-klienta.md (2026-08-31)", "vault:10_Projects/hexbane/assety-i-pipeline.md (2026-08-31)", "vault:10_Projects/hexbane/protokol-klient-serwer.md (2026-08-31)"]
---

# Legacy, tooling and known dead spots (client)

Facts preserved from the 2026-08-31 reconnaissance that no other note covers. Verified on
2026-09-07 unless marked *(unverified)*.

## Dead or legacy code

- Removed on 2026-09-08: missing-scene routes `startup`, `autoload`, `debug_rpcs`, their unused navigation methods, and the unused GTweens/Godot tween plugin/autoload.
- `Game/ScenesV3/GameHud/Main.tscn` and `GameHud/ArcaneDuel/Main.tscn` are orphans: the live match
  scene is `ReferenceDuel/MainReference.tscn` (see [[duel-v2-client]]); only `Dev/HudPreview.cs`
  references the old HUD.
- `Core/Common/Enums/Opcodes.cs` still declares the retired combat opcodes 11–15 and 21–28; the match
  dispatcher excludes them (see [[opcodes]]).
- `Core/Spells/Spell.cs` maps `mirror_reflection → mirror_ward`, `firebolt → fireball`,
  `heavy_bolt → flamestrike` as legacy asset aliases; `Core/Spells/StandardSpells.cs` cites a YAML path
  that no longer exists on the server.
- `Core/Characters/Character.cs` hard-codes `MaxSpellSlots = 6` until the server value arrives.
- The client call to unregistered `start_story` and the entire storytelling path were removed on 2026-09-08. `get_users` remains a separate pre-existing issue; see [[rpcs]].
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

Removed on 2026-09-08: `prompt.md`, `vfx.md`, `spell_output.md`, root `magic_sparkle.png`, sample images in `Scripts/workflow`, and generated primitives/textures/motions/references/JSON under `Resources/Spells`. Background-removal tools remain available for current artwork.

## Race art history

- `~/hexbane-archive/Races-2026-08-31/` (922 MB, never in git): the withdrawn 35-race ChatGPT roster,
  `_raw_chroma/`, `_style_comparison/`, and the old race bibles.
- The `race-maker` skill (`.claude/skills/race-maker/`) describes the ChatGPT chroma-sheet method
  (solid chroma per race, green `#00FF00` default, no painted glow, ~60–65 % of the cell, ~30 px
  margin, frame 1 = idle, one ChatGPT thread per race). Its helper scripts `extract_grid.py`,
  `slice_sheet_alpha.py`, `upscale_frames.jsx` are untracked in git. The shipped six races were built
  with the Blender/Mixamo pipeline instead — see [[assets-pipeline]].
- `Resources/Icons/` holds 31 legacy SVG icons; old spell and UI SFX libraries were removed on 2026-09-08; Mirror Reflection formation/shatter audio was restored afterwards; menu/battle music remains.

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

## Spell animation skill (Codex)

Personal skill: `/Users/elanon/.codex/skills/hexbane-spell-animation/SKILL.md`, with UI metadata in `agents/openai.yaml`. Invoke, for example: `$hexbane-spell-animation zrób animację Poison`.

The skill covers nature/palette research, visual intent, shared catalog/factory/actor playback, authoritative lifecycle, real ArenaMapsDev controls, GPU inspection and vault updates. It uses [[spell-effect-system]], [[spell-vfx-configuration]], [[spell-lore]] and current source code rather than copying a spell catalog. It is scoped to Hexbane spell VFX, not balance changes or standalone icons. The file is installed locally; it is not distributed with the client repository.

## Source of truth in code
- `client:Game/Autoloads/SceneManager.cs` — route table incl. dead keys
- `client:project.godot` — autoload list, `[hexbane]` settings
- `client:Application/Nakama/NakamaClientManager.cs` — server presets and host resolution
- `client:Core/Spells/Spell.cs`, `client:Core/Spells/StandardSpells.cs` — legacy id aliases
