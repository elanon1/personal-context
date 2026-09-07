---
type: project
project: Hexbane
area: audits
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
tags: [hexbane, audit, docs-verification]
---

> Audit report from the 2026-09-07 documentation consolidation (see [[10_Projects/hexbane/_state|_state]] decisions log). Discrepancies here were already applied to the merged notes; the "code smells" and "open questions" sections are the backlog. Paths: `client:` = `~/RiderProjects/hexbane`, `server:` = `~/GolandProjects/hexbane-server`.

# Vault audit: the five topic notes in `10_Projects/hexbane/` (2026-08-31) vs working trees 2026-09-07

For each note: what still holds, what is superseded (and by which staged doc), and facts that **no staged doc covers** (quoted verbatim, Polish, so they can be preserved when the vault is rewritten). Staged docs = `staging/**` in this scratchpad; `[[x]]` = staged basename.

---

## 1. `architektura-klienta.md`

### Still true
- Layering Game → Application → Core with upward reaches into `Game.Autoloads`/`DIHost` (service-locator smell). Verified indirectly: `client:Core/Match/MatchState.cs` still under `Core/` (untouched in status).
- Main scene `Game/ScenesV3/Auth/AuthScreen.tscn`, viewport 2400×1080, `canvas_items`/`expand`, `gl_compatibility` (`client:project.godot:18,47-48,52-53,100`).
- `.env` packed as `Content` (`client:hexbane.csproj:30`).
- Keyed `IMatchManager`: `ad_normal`, `ad_ai`, `create_character` (`client:Game/DI/ServiceBootstrapper.cs:107-109`).
- Nakama servers hard-coded: `local`, `synology` (:7350), `prod hexbane.elanon.pl:443` (`client:Application/Nakama/NakamaClientManager.cs:113-135`).
- `EnvLoader` keys read in code: `NAKAMA_SERVER`, `NAKAMA_HOST`, `NAKAMA_PORT` (new), `DEV_AUTO_LOGIN_MODE`, `AUTH_EMAIL`, `AUTH_PASSWORD` (grep `EnvLoader.Get(`); `DEV_AUTO_LOGIN` and `DEBUG_SCENE` are in `.env` but not seen through that accessor (unverified how they are read).
- GTweens autoload with zero users (grep `GTween` in `Game/`,`Application/` = 0 files).
- Dead SceneManager keys `debug_rpcs` (→ `Game/Scenes/Debug/RPC.tscn`, folder absent) (`client:Game/Autoloads/SceneManager.cs:32`).

### Superseded / stale
- `Godot.NET.Sdk/4.7.0` → **4.5.2** (`client:hexbane.csproj:1`). (Note was wrong even on 2026-08-31; HEAD commit says 4.5.2.)
- Autoload list: now `DIHost, DpiScaler, EnvLoader, GameContext, GameEvents, MatchContext, SceneManager, MainThreadInvoker, GodotGTweensContextNode, NotificationManager, MenuPlayer, DevAutoLogin, MCPScreenshot, MCPInputService, MCPGameInspector` (`client:project.godot:27ff`).
- Auth: email/password only → **Google sign-in via built-in `AuthenticateGoogleAsync`** plus email (`client:CLAUDE.md` "Google sign-in", staged [[google-auth]]).
- Match message flow: opcodes 11–15/21–28 retired, combat is 29–32 (staged [[opcodes]], [[combat-v2]]); the `GameHud` description (10 L/R slots, `CastSweepOverlay`, `EffectSlot`, swipe-up meditation) describes `Game/ScenesV3/GameHud/Main.tscn`, which is **no longer the match scene**: `SceneManager.cs:30` maps `normal_game` → `ReferenceDuel/MainReference.tscn` (`client:docs/client/reference-duel/README.md`).
- "Orphaned `GameHud/ArcaneDuel/Main.tscn`": still present, and `GameHud/Main.tscn` joined it as the orphan (only `Dev/HudPreview.cs` references its `Backgrounds/Sky`).
- VFX section: 9 active configs → 10 (`poison_dart` re-enabled), none matching the 14-spell catalog; new preset system `Resources/SpellVisuals/*.tres` + `visual_key` (see vault-state-audit fact 3, staged [[spell-visual-key]]).
- Design system: fonts are now Cinzel / Cormorant Garamond / Marcellus / Inter (`client:Resources/Fonts/`), not Overpass; `docs/DESIGN_SYSTEM.md` still exists (content vs fonts unverified).
- Screens: Dashboard/CreateCharacter/CharacterDetail/Lobby/GameOver/Settings/Social/News were rebuilt from `client:reference/*.png` mocks 2026-08-31..09-02 (memory notes); a Tutorial screen exists (`SceneManager.cs:356` lists `TutorialScreen`). Details belong to the client-docs agent.
- `docs/opcodes` "synchronized from server": both copies drifted; see protocol staging.

### Facts not covered by any staged doc (preserve)
> „`GameContext`: `Character`, `IsAuthenticated`, klucz serwera; timer 30 s sprawdza sesję/socket i wylogowuje.” (unverified today)
> „Socket `Closed` → reset + logout; **brak auto-reconnect**.”
> „**Martwe klucze**: `startup`, `autoload`, `debug_rpcs` (folder `Game/Scenes/` nie istnieje).” — `debug_rpcs` confirmed at `SceneManager.cs:32`.
> „**GTweens** (autoload) ma zero użyć — wszędzie natywny `Tween`.” — confirmed.
> „`FlashCaster` z CLAUDE.md nie istnieje.” — CLAUDE.md no longer mentions it; moot.
> „`docs/DESIGN_SYSTEM.md`: tła brązowo-czarne (R>G>B, alpha 0.45–0.95), akcent `#D9591F`, … promienie 8/10/12, kolory statów STR/INT/DEX = czerwono-pomarańczowy/niebieski/zielony. Agent `ui-designer` w `.claude/agents` pilnuje spójności.” (font part stale, rest unverified)
> „`EnvLoader` (`.env`): `NAKAMA_SERVER` (`local|synology|prod`), … `DEV_AUTO_LOGIN_MODE` (`existing|new_character`), `AUTH_EMAIL/PASSWORD`, `DEBUG_SCENE`. Brak `.env.example`.” — still no `.env.example`.

---

## 2. `architektura-serwera.md`

### Still true
- Go 1.24, `nakama-common v1.37`, `sqlx`, `godotenv` (fatal without `.env`), `yaml.v3`; plugin for Nakama 3.27.0 (`server:go.mod:3,8-11`, `Dockerfile:1,14`).
- Module init order in `modules/main.go`: `race → character → healthcheck → normal_match → ai_match → spellbook → playstyle → auth → social → notifications → spell_system → spell_effects → endless_story → RegisterAllHandlers` (`server:modules/main.go:37-114`). **Minus** `player (legacy)`: `modules/match/engine/player/*` deleted in working tree.
- Match names `"normal"`, `"ai_duel"`, story match `v2_create_character` (`server:modules/match/normal_match/init.go:61`, `ai_match/init.go:27`, `endless_story/create_character/init.go:25`).
- Module file convention `init.go/types.go/rpc.go/db.go/validate.go` (spot-checked, unverified for every module).
- `docs/plans/level-up-notifications.md` still unimplemented (staged [[level-up-notifications]]).

### Superseded (→ staged doc)
- Spell catalog 63 files in `data/spells/<school>/<tier>/` → 14 flat files `data/spells/*.yaml` → [[spell-system]].
- Migrations `000001..000010` + planned `000011_clear_races` → fresh `000001..000003` → [[database]], vault-state-audit.
- `race.DefaultRaceId = ""` → `"human"` (`server:modules/race/types.go:6`).
- `spells` table "unused at runtime" → table no longer exists (`server:db/migrations/000001_initial_schema.up.sql` creates `races, characters, character_spells, playstyles, playstyle_slots`; `000003` adds `account_tutorials`).
- RPC list (29) → [[rpcs]] (adds `tutorial`, `get_character_details`, removes/marks broken others).
- Phase timings (LobbyCountdown 15 s, GameCountdown 2 s, Combat 300 s), combat tick contents (op11/12/13/14/15/22/24 cadence), cast formula with DEX/race, meditation `rand(10..30)`, passive regen, 100 % reflection, bot 150 HP/150 mana, AI priority tree → all replaced by duel_v2: [[server-architecture]], [[progression]], [[combat-v2]], [[matchmaking]]. Bot now takes per-race base stats (`server:modules/match/ai_match/bot.go:36-43`) and a random race.
- Effect types (11 with 4 unhandled) → 11 new kinds, all handled (vault-state-audit fact 2).
- `EffectQueue` keying/stacking → [[spell-system]] "Effect queue semantics".
- "GUIDE-v2 describes archival prototype (`bkp/`)" → GUIDE-v2 is a stub; `bkp/` deleted in working tree.
- Docs list: `docs/client/effect-sync.md` is a stub; `docs/opcodes/*` 14 modified, still has `op_09_game_countdown.md`; `docs/devlog.md` unchanged.
- Endless Story: still as described (only `create_character` match + `story_model/`), plus `start_story` still commented — unchanged but flagged OPEN in vault-state-audit.

### Facts not covered by any staged doc (preserve)
> „Konwencja modułu: `init.go` (rejestracje), `types.go`, `rpc.go`, `db.go`, `validate.go`.”
> „**Zakomentowane**: `start_story`, `create/continue_character_story`, 13× `social_*` (status/follow/chat na wbudowanych API Nakamy).” — `start_story` confirmed (`endless_story/init.go:13`); the 13 `social_*` stubs: staged [[social]] has a "Dropped from the old docs" section, count unverified.
> „Stan: `MatchState` (RWMutex „defensywny” — jeden goroutine na mecz), … `MatchStats` nigdy nie wypełniany; `TransitionTo` omijane przez większość faz” — `MatchStats`/`TransitionTo` mentioned in staged [[server-architecture]]; the "one goroutine per match" rationale for the mutex is not.
> „2025-08-12 — Serwer: model „mecz prowadzony przez jednego gracza” zamiast czystych RPC dla tworzenia postaci (Endless Story). Why: zarządzanie sesją/pamięcią przez Nakama match zamiast trzymać stan w RPC.” (decision rationale, from `_state.md`)

---

## 3. `protokol-klient-serwer.md`

### Still true
- Opcode numbering identical on both sides; sources of truth `client:Core/Common/Enums/Opcodes.cs`, `server:modules/match/engine/match_types/op_codes.go`.
- Opcodes 0–10, 16, 50, 70, 100/101, 199 keep their numbers and roles (staged [[opcodes]] index; `client:Opcodes.cs:8-56`).
- Nakama match names vs DI keys distinction (`"normal"`/`"ai_duel"` vs `ad_normal`/`ad_ai`) — still true and now in staged [[matchmaking]].
- RPC mapping rows for character/race/spellbook/spell/social/notifications: still valid, extended in [[rpcs]] (`tutorial`, `get_character_details`).
- `start_story` still called by client and still commented on the server (both verified, see vault-state-audit fact 6). `get_users` (`client:Application/Modules/Social/Queries/GetUsers/GetUsersQueryHandler.cs`) still has no server RPC ([[rpcs]] "Client ↔ server coverage").

### Superseded (→ staged doc)
- Rows 11, 12, 13, 14, 15, 21–28 (combat): **retired**; combat is 29 `COMBAT_COMMAND`, 30 `COMBAT_RESULT`, 31 `COMBAT_EVENT`, 32 `COMBAT_SNAPSHOT` (`client:Opcodes.cs:44-47`; staged [[combat-v2]], [[op_29_combat_command]]…[[op_32_combat_snapshot]]). The enum still *declares* 11–15 and 21–28 (`client:Opcodes.cs:29-42`), so they are dead constants on the client.
- `PlayerSnapshot` field list, `SpellStatus`, `SpellFailed` reasons → replaced by duel_v2 snapshot / `action_rejected` reasons ([[combat-v2]], [[shared-types]]).
- Payload of op 0 now also carries `combat_protocol`, `ruleset_id`, `catalog_version`, `tick_ms`; op 2 must send `combat_protocol: 2` (staged `server/architecture.md:98`).
- "Effect sync rules from `docs/client/effect-sync.md`" → that doc is a stub; rules now: effect instance ids + tick deadlines from opcode 32 ([[combat-v2]]).
- "GameCountdown: docs say 9, code 16" → resolved in staging as [[op_16_game_countdown]]; the stale `op_09_game_countdown.md` files still exist in both repos (report item).

### Facts not covered by any staged doc (preserve)
> „Serwer ma też RPC, których klient nie woła: … `get_spell_details_yaml` (używane przez n8n/`vfx.md`)” — the **n8n consumer** of `get_spell_details_yaml` is not mentioned in any staged doc (only the RPC itself in [[rpcs]]/[[spell-system]]).
> „26/27/28 MeditationFailed/Accepted/Interrupted … klient: puste handlery” — historical; retired.

---

## 4. `zasady-gry.md`

### Still true
- Character creation: 400 points over STR/INT/DEX, name 3–20, one character per account — partially; now **3 starter spells (Human: 4)** not "exactly 4" (`server:RPCs.md:3` banner; staged [[rpcs]] `create_character`), race stat floors/ceilings apply (staged `server/progression.md` "Races").
- XP/levels/magic points/skill-gain formulas exist as progression data — keep only via staged [[progression]] "XP and levels", "Magic points", "Skills" (values there are verified; the vault's `+50 daily bonus`, `1.5^(level−2)`, MP tiers etc. must be re-read from that doc, not from the note).
- Lobby draft 35 s per player, alternating, auto-fill on timeout (staged `server/architecture.md:99`); client arrange-mode `SpellSlotOrder` (client-side, not covered by staged server docs — see below).
- Spell descriptions in Polish — **no longer**: `server:data/spells/firebolt.yaml:3` `description: Deals 16 direct damage.` (English).

### Superseded (→ staged doc)
- `MaxHealth = STR`, `MaxMana = INT`, DEX cast/dodge/regen scaling, skill effect `skill/200`, damage/heal/resistance formulas, `mind` hard-coded damage type, poison flat ticks, shield overwrite, paralysis broken by damage, 100 % reflection, passive regen, meditation `rand(10..30)…`, 300 s match, bot 150/150 with priority tree → **all inactive or replaced** in `duel_v2`: fixed 200 HP / 100 mana, 180 s limit, catalog values, "none of the stored stats, skills or racial combat traits change a duel" (staged `server/progression.md:16,113`, [[combat-v2]]).
- Catalog "63 = 9 schools × 7 tiers" with tier cost table → 14 spells, per-spell `mana_cost`, `casting_time`, `recovery_time`, `travel_time`, `standard`, `starter` flags (`server:data/spells/firebolt.yaml:9-16`; staged [[spell-system]]).
- Effect types with 4 missing handlers and the 18 affected spell ids → gone (vault-state-audit fact 2).
- Balance trivia (`mirror_ward` 120 s reflection, `discharge` 54, `paralyze` 10 s) → those spell ids no longer exist.
- Learning spells "MP cost default 5, tiers 1–2 = 0" → `magic_point_cost` per YAML (firebolt: 5); tiers gone.
- "`data/spells - spells.csv` design sheet not read by code" → file deleted in working tree.

### Facts not covered by any staged doc (preserve)
> „Klient dodatkowo pozwala **ułożyć** wybrane zaklęcia w lewą/prawą rękę (`SpellSlotOrder`).” — client lobby "Arrange Your Spells" mode; no staged server doc mentions it (client-docs agent's area; grep `SpellSlotOrder` in staging = none).
> „Opisy zaklęć są po polsku (`description`)” — now false, but worth recording that the catalog language flipped to English in the redesign.

---

## 5. `assety-i-pipeline.md`

### Still true
- What the code expects from a race folder: `avatar.png`, `front.png` (paths in `client:Core/Characters/Character.cs`, unverified line), `animation/frames.tres` with `idle` for `RaceAnimationPreview`; server row in `races` with `race_id` = folder name; roster from RPC `get_races`. Extended: `frames_sd.tres`, 16 clips, `hand_tracks.tres`, `meditation_tracks.tres` (staged [[assets-pipeline]]).
- `Resources/Spells/` old-prototype folders with `spell.json` + `manifest.generated.json` (10 of them) from the n8n pipeline; `Resources/Icons/` 31 legacy SVG; almost no audio for spells (`Resources/Sound/{fireball.wav,heal.mp3}`).
- n8n prompt files (`prompt.md`, `vfx.md`, `spell_output.md`) and `.claude/commands/create_spell_*` — files still present (`spell_output.md` is even csproj Content, `client:hexbane.csproj:38`); whether the n8n workflow still runs: unverified.
- `env-concept` / `env-maker` skills and `Resources/Environments/` exist (3 arenas now).
- Addons `godot_mcp`, `ColorPreview`, `rider-plugin` (+ new `hexbane_android` (enabled, required), `GodotPlayGameServices` (disabled)). `.claude/`: 7 agents, **34** commands, 3 skills; `MAX_THINKING_TOKENS=32000` (`client:.claude/settings.json:11`).
- `remove_bg.py` (rembg) in `Scripts/` with `uv`.

### Superseded (→ staged [[assets-pipeline]])
- "Roster empty since 2026-08-31; only `_tools/{key_sheet.py, build_race_frames.py}` left" → six races on disk, built by a Blender/Mixamo pipeline; `_tools/` now has `build_races.sh`, `build_frames_from_renders.py`, `pack_meditation.py`, `export_hand_tracks.py`, `blender/*`.
- "Complete race format per `race-maker`" (ChatGPT 3×3 chroma sheets, 6 anims) → describes the skill, not the shipped assets. The shipped clips are 13 Mixamo clips + 3 meditation clips rendered from a rigged Tripo FBX.
- "Duel does not render races" → false (vault-state-audit fact 1).
- "Active match scene uses `Resources/Backgrounds/Sky/` + TileMapLayer" → live scene is `ReferenceDuel/MainReference.tscn` with three arena PNGs from `Resources/Images/Arenas/` (`client:Game/ScenesV3/ReferenceDuel/ArenaCatalog.cs:14-16`) and shaders; `Resources/Environments/*` parallax sets are referenced only by dev scenes (`Game/ScenesV3/Dev/ArenaMaps/*`).
- Spell coverage numbers (9 of 63, 19 folders) → 21 folders, 2 of 14 catalog ids have an icon folder (`magic_arrow`, `mirror_reflection`), VFX presets for 4 ids in `Resources/SpellVisuals/`.
- `deploy.sh` Windows/WSL2 → Mac (vault-state-audit fact 7).

### Facts not covered by any staged doc (preserve)
> „Stara sztuka **w archiwum** `~/hexbane-archive/Races-2026-08-31/` (922 MB: 35 folderów ras + `_raw_chroma/` + `_style_comparison/` + `_server-docs-client-races/` ze starymi biblami 8 ras i tumbloama). Nigdy nie było tego w gicie.” — archive dir still exists (listed 2026-09-07); contents not re-verified. Mentioned only briefly in [[assets-pipeline]].
> „Pipeline: tło = jednolita chroma per rasa (zielona `#00FF00` domyślnie, magenta/niebieska gdy paleta koliduje), **zero malowanych efektów/glow** (VFX dodaje silnik), figura ~60–65 % komórki, ~30 px marginesu, klatka 1 = Idle; jeden wątek ChatGPT na rasę, idle jako referencja.” — the ChatGPT chroma recipe; lives in `race-maker/SKILL.md`, summarized as "legacy path" in [[assets-pipeline]].
> „Skrypty: `.claude/skills/race-maker/scripts/{extract_grid.py, slice_sheet_alpha.py, upscale_frames.jsx, ps_cutout.jsx}`.” — first three are **untracked** in git (`git status .claude`), `ps_cutout.jsx` not seen in status (tracked or absent, unverified).
> „10 z pełnym pipeline'em (`spell.json` + `manifest.generated.json` + `primitives/textures/motions/references`, linki Google Drive z n8n)” — the Google-Drive-links detail.
> „Prompty n8n: `prompt.md` (zaklęcie → JSON z promptem ikony i audio), `vfx.md` (definicja → `vfxBlueprint`; woła `get_spell_details_yaml` na Nakamie), `spell_output.md` (schemat `spell.json`).”
> „Addony: `godot_mcp` (Godot MCP Pro 1.14.1, WebSocket :6505, serwer Node w `mcp_server/`)” — version/port unverified today.
> „Gotchas MCP (scena aktywna vs `create_scene`, tekstury przez `execute_editor_script`, tracki animacji) — w pamięci Claude Code.”
