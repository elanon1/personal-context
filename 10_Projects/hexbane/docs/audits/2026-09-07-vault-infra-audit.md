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

# Report: vault notes audit + assets pipeline + repo snapshot (area `infra`)

Scope: the seven vault notes in `10_Projects/hexbane/` (2026-08-31), `client:Resources/**`, `client:.claude/skills/**`, both `.git` trees. Read-only; outputs in `staging/infra/` and `reports/`.

## 1. Input → verdict

| Input | Verdict |
|---|---|
| `vault:_state.md` | audited claim-by-claim → `reports/vault-state-audit.md`; not merged (it is a state note, to be rewritten by the lead from the audit) |
| `vault:architektura-klienta.md` | superseded by client staging (client-docs agents) + [[infra-and-deploy]]; 7 uncovered facts quoted in `reports/vault-notes-audit.md` §1 |
| `vault:architektura-serwera.md` | superseded by [[server-architecture]], [[spell-system]], [[database]], [[rpcs]], [[matchmaking]]; 4 uncovered facts quoted §2 |
| `vault:protokol-klient-serwer.md` | superseded by [[opcodes]], [[combat-v2]], [[rpcs]], [[shared-types]]; 1 uncovered fact (n8n consumer of `get_spell_details_yaml`) §3 |
| `vault:zasady-gry.md` | superseded by [[progression]], [[spell-system]], [[combat-v2]]; nearly every number is dead under duel_v2; 2 uncovered facts §4 |
| `vault:assety-i-pipeline.md` | merged into `staging/infra/assets-pipeline.md` (rewritten from the working tree); 7 uncovered facts quoted §5 |
| `vault:infra-i-deploy.md` | already merged by the infra agent into [[infra-and-deploy]]; cross-checked here (see discrepancies) |
| `client:Resources/Races/_tools/build_races.sh`, `_tools/blender/README.md`, `_tpose/README.md`, `*/PLACEHOLDER.md` | merged into `assets-pipeline.md` |
| `client:.claude/skills/{race-maker,env-concept,env-maker}/SKILL.md` | summarized in `assets-pipeline.md` (race-maker marked legacy route); not documentation of the repo state, so not copied |
| `client:Resources/SpellVisuals/README.md` | merged (summary) into `assets-pipeline.md` |
| `client:Resources/Images/Arenas/PROMPTS.md`, `Images/ReferenceDuel/PROMPTS.md` | merged (summary) into `assets-pipeline.md` |
| `client:Resources/Environments/*/environment.json` | merged (schema summary) into `assets-pipeline.md` |
| git metadata of both repos | → `staging/infra/repos-and-branches.md` |

## 2. Discrepancies found

### Vault `_state.md` / notes vs working trees (headline items; full table in `vault-state-audit.md`)
- Vault: 63 spells → code: **14** (`server:data/spells/*.yaml`; 63 old files deleted).
- Vault: 35 races, roster empty client-side, `race.DefaultRaceId = ""` → code: **6 races** (`server:db/migrations/000002_reference_data.up.sql:1`; `client:Resources/Races/{dark_elf,elf,gnome,human,orc,shadow}`), `DefaultRaceId = "human"` (`server:modules/race/types.go:6`).
- Vault: duel renders a generic `Skeleton2D` from `Resources/Chars/Concept1` → code: `RaceSpriteAnimator` in the live duel (`client:Game/ScenesV3/ReferenceDuel/ReferenceHud.cs:510-532`, `client:Game/ScenesV3/GameHud/ArcaneDuel/Components/Player.cs:26`); `Concept1` has zero references.
- Vault: 4 effect types without handlers → code: 11 kinds, all registered (`server:modules/spell_system/types.go:15-25`, `.../effect_handlers/registry.go:5-16`); `stun/slowdown/absorb/mana_drain` no longer exist.
- Vault: `GameCountdown` docs 9 / code 16 → code 16 on both sides (`server:modules/match/engine/match_types/op_codes.go:24`, `client:Core/Common/Enums/Opcodes.cs:24`); stale `op_09_game_countdown.md` still present in **both** `docs/opcodes/` folders.
- Vault: secrets present in `.env.dist`, `helm/values.yaml`, client `.env` → **all still present** (`server:.env.dist:4`, `server:helm/hexbane/values.yaml:15`, `client:.env` tracked with `AUTH_PASSWORD`/`GOOGLE_CLIENT_SECRET` set). New: `client:project.godot:70-71` Google Desktop-app id/secret (by design), `client:hexbane.csproj:31` lists `client_secret_*.json` as Content.
- Vault: `endless_story` registered, `start_story` commented, client still calls it → **unchanged** (`server:modules/main.go:109`, `server:modules/endless_story/init.go:13`, `client:Application/Modules/Story/Query/StartStory/GetSpellQueryHandler.cs:36`).
- Vault: `deploy.sh` Windows/WSL2 → now Mac paths (`client:deploy.sh:7-8`); `hexbane.csproj:5` `StartProgram` still `F:\Godot\...`.
- Vault: migration `000011_clear_races` to add manually → never created; tracked set is `000001..000009, 000012..000016` (28 files, gap at 10–11), working tree replaces it with `000001..000003`.
- Vault: k8s prod described in `infra-i-deploy.md` → still only in `server:helm/**` and `server:deploy/argocd/application.yaml`; **no** doc in either repo mentions Argo/Helm/`elanon.pl` any more (grep `server:docs/*.md`, `README.md` = none).
- Vault: CI "no tests/lint before push" → CI now has a `test` job (`go test ./...`, `go test -race` on match/spell_system/duel-sim) gating the image build (`server:.github/workflows/docker-publish.yml:18-31`).
- Vault: Android package `com.example.$genname` → `pl.elanon.hexbane` (`client:export_presets.cfg:110`).
- Vault: `Godot.NET.Sdk/4.7.0` → `4.5.2` (`client:hexbane.csproj:1`); was already wrong when written.
- Vault: client branch `master`, server branch `main` → checked-out `feat/duel-v2-client` and `feat/spell-system-redesign`, both equal to their default branch commit; all redesign work uncommitted.
- Vault: match 300 s, bot 150 HP/150 mana, `MaxHealth = STR`, DEX/skill formulas → duel_v2: 180 s, fixed 200 HP / 100 mana, stats/skills/race traits inactive (staged `server/progression.md:16,113`).
- Vault: spell descriptions in Polish → English (`server:data/spells/firebolt.yaml:3`).
- Vault: fonts Overpass (+Cinzel Decorative) → `Resources/Fonts/{Cinzel, Cormorant_Garamond, Inter, Marcellus}`.
- Vault: active match scene uses `Resources/Backgrounds/Sky` + TileMapLayer → `ReferenceDuel/MainReference.tscn` with `Resources/Images/Arenas/*.png` (`client:Game/ScenesV3/ReferenceDuel/ArenaCatalog.cs:14-16`, `client:Game/Autoloads/SceneManager.cs:30`).
- Vault: `.claude/` has 30 commands → 34; addons list lacks `hexbane_android` (required) and `GodotPlayGameServices` (disabled).
- Vault: client VFX 9 active configs → 10 (`poison_dart` back), still 0 of 14 catalog ids (`client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs:14-40`).

### Staged docs vs what I measured (for the lead to reconcile)
- `staging/infra/infra-and-deploy.md` "Local only" says the working tree deletes tracked `000001..000016`. Tracked set is actually `000001..000009` and `000012..000016` (28 files; `000010`/`000011` were never committed, `git ls-files db/migrations`). Same conclusion (prod DB is at 16), wording slightly off.
- `staging/server/spell-system.md` / brief say catalog `duel_v2.2`; `server:docs/spell_system/spells.md:1` still titles itself `duel_v2.1`. Not verified which the code's `version.go` says (spells agent's area; flagging only).
- `client:CLAUDE.md` says the APK is ~670 MB; the current `hexbane1.apk` in the repo root is ~1.05 GB (untracked; probably includes both sprite sets or a newer build — unverified why).
- `blender/README.md:185` says `Player.tscn` "no longer carries the old Skeleton2D rig" and describes clip selection; `Player.cs` matches, but the live scene wiring goes through `ReferenceHud.cs`, which constructs `RaceSpriteAnimator` in code (`:510`). Both paths exist; the doc describes the `GameHud/ArcaneDuel` one.

## 3. Code smells / dead code noticed
- `client:Resources/Chars/Concept1/` — unreferenced (old rig art).
- `client:Game/ScenesV3/GameHud/Main.tscn` and `GameHud/ArcaneDuel/Main.tscn` — no longer the match scene; `Resources/Backgrounds/*` only used by them and `Dev/HudPreview.cs`.
- `client:Core/Common/Enums/Opcodes.cs:29-42` — retired opcodes 11–15, 21–28 still declared.
- `client:Application/Modules/Story/**` (`StartStoryQuery` → `start_story`) and `client:Game/DI/ServiceBootstrapper.cs:109` keyed `create_character` match manager — call a commented-out server RPC; `server:modules/endless_story` still registered.
- `client:Application/Modules/Social/Queries/GetUsers/*` — no server RPC `get_users`.
- `client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs` — 20 configs for spell ids that no longer exist server-side; `Game/FX/*.tscn` (21) mostly orphaned by the catalog.
- `client:Resources/Spells/` — 19 of 21 folders are old ids; 10 carry n8n `spell.json` bundles for a dead pipeline. `Resources/Icons/` 31 legacy SVGs.
- `client:Resources/Environments/*` (3 parallax sets, env-maker output) — used only by dev scenes; the duel uses `Images/Arenas`.
- `client:.claude/skills/race-maker` + `_tools/key_sheet.py`, `_tools/build_race_frames.py` — describe the abandoned ChatGPT chroma route; `race-maker/scripts/*` untracked.
- `client:hexbane.csproj` Content items: `.env`, `client_secret_*.json`, `magic_sparkle.png`, `reference\fight\game.png`, `spell_output.md`, `unnamed.jpg`, `Scripts\remove_bg.*` — junk shipped into builds (`:30-39`). `StartProgram` Windows path (`:5`). `hexbane.csproj.old.1..5` in repo root.
- `client:hexbane1.apk` (~1.05 GB) untracked in repo root; `create/`, `screen/`, `unnamed.jpg` scratch files.
- `client:Game/Autoloads/SceneManager.cs:32` dead key `debug_rpcs` → non-existent `Game/Scenes/Debug/RPC.tscn`.
- GTweens autoload (`GodotGTweensContextNode`) with zero users.
- `server:docs/opcodes/op_09_game_countdown.md` and `client:docs/opcodes/op_09_game_countdown.md` — stale filename for opcode 16.
- `server:.env.dist` `OPENAI_*`, `PLUGIN_PATH` and `helm` `OPENAI_*` env — unused by code, still carrying a key.
- `server:helm/hexbane/values.yaml:15` PAT inline; `database.password` placeholder.
- `server:RPCs.md` — banner says superseded, body still old examples.
- `client:.env` tracked **and** listed as csproj Content: dev password ships in desktop builds (Android drops dotfiles, per `client:CLAUDE.md`).

## 4. Open questions (need a human)
1. **Endless Story**: delete `server:modules/endless_story` + client `Modules/Story` + keyed `create_character` manager, or rewrite without OpenAI? Code has sat half-removed since 2026-08-31.
2. **Prod deploy strategy**: the fresh migration set (`000001..000003`) cannot be applied on top of a prod DB at version 16 without a reset. Is prod (`hexbane.elanon.pl`) still alive/wanted, and is wiping it acceptable? Until answered, nothing from the redesign can go to `main`.
3. **Secrets**: rotate the OpenAI key (`server:.env.dist:4`), the GitHub PAT (`server:helm/hexbane/values.yaml:15`), and the dev login (`client:.env`), and purge from history? Still not done since 2026-08-31.
4. **What to commit of `Resources/Races/`**: the sheets + `.tres` (intended, per `.gitignore`), and also `Resources/Races/3d/` (112 MB FBX/blend) and `_tpose/`? Git LFS or archive?
5. **Which arena system is canonical**: `Resources/Environments/*` parallax sets (env-maker, unused in the duel) or the three `Images/Arenas/*.png` + shader approach the live duel uses? The env skills produce assets nothing ships.
6. **Spell art for the 14-spell catalog**: only `magic_arrow` and `mirror_reflection` have any client asset. Is the n8n icon/VFX/SFX pipeline (`prompt.md`, `vfx.md`, `get_spell_details_yaml`) still meant to run, or should `Resources/Spells/` old folders be archived like the races were?
7. `hexbane.csproj` Content list and the five `hexbane.csproj.old.N` files: safe to clean? (`.env` as Content is the security-relevant one.)
