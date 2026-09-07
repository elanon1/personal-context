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

# Vault audit: `10_Projects/hexbane/_state.md` (written 2026-08-31) vs working trees on 2026-09-07

Legend: **TRUE** = still holds · **STALE** = contradicted by current working tree · **PARTLY** = core holds, detail wrong · **DONE** = the open item was resolved.
Line refs are working-tree (uncommitted) state. `client:` = `~/RiderProjects/hexbane`, `server:` = `~/GolandProjects/hexbane-server`.

## Summary

| Claim | Verdict | Evidence |
|---|---|---|
| 1v1 real-time mage duel, no movement, cast time, meditation for mana | TRUE | `server:modules/match/engine/phase/game/phase.go` (see staged [[progression]] "Combat rules"); staged `server/progression.md:113` |
| Client repo `elanon1/hexbane`, branch `master` | PARTLY | remote `git@github.com:elanon1/hexbane.git`; checked-out branch is `feat/duel-v2-client`, which points at the same commit as `master` (`01ec454`, 2026-08-01; `git rev-list --count master..HEAD` = 0). All work since is uncommitted. |
| Client: Godot 4.5 + C# .NET 9, Nakama SDK 3.16, Game→Application→Core, DI + CQRS | TRUE | `client:hexbane.csproj:1` `Godot.NET.Sdk/4.5.2`, `:3` `net9.0`, `:49` `NakamaClient 3.16.0`; `client:project.godot:20` features `4.5, C#, Mobile` |
| Server repo `elanon1/hexbane-server`, branch `main` | PARTLY | remote correct; checked-out branch is `feat/spell-system-redesign` = `main` = `017ed09` (2026-09-05). Redesign is entirely uncommitted (276 status entries). |
| Server: Go plugin for Nakama 3.27 (`backend.so`), Postgres 17 | TRUE | `server:Dockerfile:1,14` (`nakama-pluginbuilder:3.27.0`, `nakama:3.27.0`); `server:docker-compose.yml:4` `postgres:17.6-alpine`; `server:go.mod:3` go 1.24.3 |
| Authoritative phased matches, 10 ticks/s | TRUE (re-based) | still 10/s, now expressed as a 100 ms logical clock, protocol 2 (`server:modules/match/engine/state/state.go:12`, staged `server/architecture.md:84`) |
| **63 spells in YAML** | STALE | 14 files in `server:data/spells/*.yaml` (barrier, cleanse, consume_venom, delayed_hex, dispel, firebolt, greater_heal, heavy_bolt, magic_arrow, mend, mirror_reflection, paralysis, poison, regeneration); the 63 old files are deleted in the working tree (`git status`: 63 `D data/spells/...`). Catalog `duel_v2.2` per brief; `server:docs/spell_system/spells.md:1` still says `duel_v2.1` (see report). |
| RPG progression: **35 races**, STR/INT/DEX, 3 skills, XP/levels/magic points | STALE (races) / PARTLY (rest) | 6 races: `server:db/migrations/000002_reference_data.up.sql:1` inserts `human, elf, dark_elf, shadow, gnome, orc`; client `Resources/Races/{dark_elf,elf,gnome,human,orc,shadow}`. Stats/skills/XP still stored but **do not affect a duel** under `duel_v2` (staged `server/progression.md:16`). |
| Prod on own k8s via Helm + Argo CD, `hexbane.elanon.pl` / `hexbane-console.elanon.pl` | PARTLY | Chart and Application still exist: `server:helm/hexbane/values.yaml:62,65` (hosts), `server:deploy/argocd/application.yaml:12-34`. Whether the cluster is running/reachable: **unverified** (no probe done). The redesign has not been deployed: CI publishes only from `main` (`server:.github/workflows/docker-publish.yml:5`) and nothing is committed. |
| 2025-06-26 decision: client rewritten GDScript → C# | TRUE (history) | no contradiction; `.gd` remains only in addons |

## Status

| Claim | Verdict | Evidence |
|---|---|---|
| Core loop works end-to-end: auth → create character → matchmaking (PvP/bot) → lobby draft → duel → game over with XP | TRUE in shape, details changed | Flow now: Google or email auth (`client:Application/Authentication/LoginService.cs`), local tutorial (`server:docs/tutorial.md`, staged [[server-tutorial]]), draft with 3–6 slots (+1 Human) and 2 standard spells (`server:docs/progression/race.md:16`), combat protocol 2 (opcodes 29–32, `client:Core/Common/Enums/Opcodes.cs:44-47`). |
| Lobby draft 35 s per turn, alternating | TRUE | staged `server/architecture.md:99` (`LobbyPickingDurationTicks = 35` s) |
| **5-minute duel** | STALE | 180 s safety limit under duel_v2 (`server:modules/match/engine/phase/game/phase.go:27`, staged `server/progression.md:113`) |
| Last devlog entry 2025-12-31; last server commits "elemental race bonuses + fix effect phase" | PARTLY | `server:docs/devlog.md` unchanged in working tree (no date grep hit in `YYYY-MM-DD` form; unverified last entry). Server HEAD is `017ed09 fix effect phase` (2026-09-05) preceded by 9 race/spell commits (see [[repos-and-branches]]). |
| Client last commits: SDK 4.5.2, race assets, login screen | TRUE | `client:git log -1` = `01ec454 Update to SDK 4.5.2; enhance race-specific assets, login automation, and UI design.` (2026-08-01) |
| 2026-06 mass production of 35 races via `race-maker`; **2026-08-31 whole roster withdrawn, `Resources/Races/` empty except `_tools/`** | STALE | `client:Resources/Races/` now holds 6 race folders + `3d/`, `_tools/`, `_tpose/` (2.5 GB on disk, of which only 2 `_tpose` files are git-tracked; everything else untracked/added). Art comes from a **Blender/Mixamo** pipeline, not `race-maker` (`client:Resources/Races/_tools/build_races.sh:1-4`). Archive `~/hexbane-archive/Races-2026-08-31/` still exists. |
| Recon doc `thoughts/shared/research/2026-08-31-hexbane-client-server-recon.md` | TRUE (file exists) | `client:thoughts/shared/research/2026-08-31-hexbane-client-server-recon.md` (untracked); content is pre-redesign, treat as historical |

## "To do manually": migration `000011_clear_races`

**OBSOLETE.** No `000011` exists. The working tree deletes all 28 tracked migration files (`000001..000009`, `000012..000016`; `000010`/`000011` were never committed) and adds an untracked fresh set `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial` (`server:db/migrations/`). Consequence for the vault's warning still applies, in a new form: pushing this set to `main` makes Argo run `migrate-custom` (`server:helm/hexbane/templates/deployment.yaml:39`) against a prod database whose golang-migrate version is ≥16 (see open questions).

## "Key facts before planning" 1–7

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Duel does not render races; player is a generic `Skeleton2D` rig from `Resources/Chars/Concept1`; `RaceAnimationPreview` reads `frames.tres` | STALE (first half) / TRUE (second) | `client:Game/ScenesV3/Components/RaceSpriteAnimator.cs:19` is the in-match sprite; used by the live duel HUD `client:Game/ScenesV3/ReferenceDuel/ReferenceHud.cs:510,523,532` and by `client:Game/ScenesV3/GameHud/ArcaneDuel/Components/Player.cs:13,26`. `SceneManager.cs:30` maps `normal_game` → `ReferenceDuel/MainReference.tscn`, `:567-569` `InMatch → GoToNormalGame()`. `Resources/Chars/Concept1` has **no references** in `Game/` (grep) → dead asset. `RaceAnimationPreview.GetRaceFramesPath` still the loader (`RaceAnimationPreview.cs:37-53`), now with HD/SD variants. |
| 2 | 4 of 11 effect types lack handlers (`stun`, `slowdown`, `absorb`, `mana_drain`); ~18 of 63 spells partly dead | STALE | Effect kinds are now 11 *different* ones: `damage, heal, poison, reflection, shield, paralyze, cure, delayed_hex, regeneration, dispel, consume_venom` (`server:modules/spell_system/types.go:15-25`); **all 11 registered** in `server:modules/spell_system/spell_effects/effect_handlers/registry.go:5-16`. `stun/slowdown/absorb/mana_drain` no longer exist. |
| 3 | Client VFX exists for 9 spells, 3 of them old prototype ids; 54 server spells have no VFX | STALE numbers, problem persists | `client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs:14-40`: 10 active (`magic_sparkle, heal, restore, cure, poison_dart, flamestrike, gust, stoneguard, mirror_ward, spark`), 10 commented. **0 of the 14 catalog ids** appear there. New path: `client:Resources/SpellVisuals/{magic_arrow,mirror_reflection,fireball,magic_reflection}.tres` presets read by `RaceSpriteAnimator.BeginCast` (`client:Resources/SpellVisuals/README.md`), plus server `visual_key` (staged [[spell-visual-key]]). `Game/FX/` = 21 scenes (old ids + `MirrorReflection.tscn`). |
| 4 | Docs behind code: GUIDE-v2 archival; `docs/opcodes` says GameCountdown=9, code 16; RPCs.md stale; race docs say 36 vs 35 | PARTLY | GameCountdown = 16 confirmed on both sides (`server:modules/match/engine/match_types/op_codes.go:24`, `server:modules/match/engine/phase/game_countdown/phase.go:17`, `client:Core/Common/Enums/Opcodes.cs:24`); stale file `op_09_game_countdown.md` still present in **both** `docs/opcodes/` dirs. `server:docs/spell_system/GUIDE-v2.md` and `docs/client/effect-sync.md` are now 3-line "superseded" stubs. `server:RPCs.md:3` carries a duel_v2 banner but old examples. Race docs now say six (`server:docs/progression/race.md:3`). |
| 5 | Secrets in repo: server `.env.dist` (OpenAI key), `helm/hexbane/values.yaml` (GitHub PAT), client `.env` (dev login, packed as `Content`) | TRUE, still present | `server:.env.dist:4` `OPENAI_API_KEY=sk-…` **yes**; `server:helm/hexbane/values.yaml:15` `password: ghp_…` **yes**; `client:.env` is git-tracked (`git ls-files .env`) with `AUTH_PASSWORD` and `GOOGLE_CLIENT_SECRET` set **yes**; `client:hexbane.csproj:30` `<Content Include=".env" />`. New since: `client:project.godot:70-71` holds the Google Desktop-app id/secret (by design per `client:CLAUDE.md`), and `client:hexbane.csproj:31` lists `client_secret_*.json` as Content. Values deliberately not reproduced here. |
| 6 | Endless Story: OpenAI code removed server-side; client still calls `start_story` (server has it commented out) | TRUE | `server:modules/endless_story/init.go:13-14` (`//RegisterRpc("start_story"…`), module still registered `server:modules/main.go:11,109`; match `v2_create_character` registered `server:modules/endless_story/create_character/init.go:25`. Client: `client:Application/Modules/Story/Query/StartStory/GetSpellQueryHandler.cs:36` `RpcAsync(session, "start_story", "")`; keyed `create_character` match manager `client:Game/DI/ServiceBootstrapper.cs:109`. |
| 7 | `deploy.sh` and csproj `StartProgram` point at Windows/WSL2 paths, not this Mac | STALE (deploy.sh) / TRUE (csproj) | `client:deploy.sh:7-8` `GODOT=/Applications/Godot.app/Contents/MacOS/Godot`, `ADB=~/Library/Android/sdk/platform-tools/adb`; `client:hexbane.csproj:5` `<StartProgram>F:\Godot\Godot_v4.4.1-stable_mono_win64_console.exe` (harmless). |

## Decisions log

Historical entries stay valid as history. Two are superseded in substance:

- **2026-08-31 roster withdrawn / new races from scratch** — done: six races (see fact 1); `race.DefaultRaceId` is now `"human"` (`server:modules/race/types.go:6`), not `""`. `RaceTest/*` and `Dev/VitraelPreview.*` are gone; new dev scenes are `client:Game/ScenesV3/Dev/{RaceAnimTest,MeditationTest,MeditationVfxPreview,ArenaMaps/...}`.
- **2026-06 race production via `race-maker` (ChatGPT chroma sheets, 6×3×3)** — the skill still exists (`client:.claude/skills/race-maker/SKILL.md`) but the shipped roster was built with Blender + Mixamo (`client:Resources/Races/_tools/blender/README.md:1`); `key_sheet.py` / `build_race_frames.py` are the legacy chroma tools, `build_frames_from_renders.py` the current one.
- **2026-06-18 sprite sheets, not bone rig** — confirmed and extended: `frames.tres` now holds 16 clips incl. meditation enter/loop/exit, plus `hand_tracks.tres` / `meditation_tracks.tres` bone tracks for VFX anchoring (`client:Resources/Races/human/animation/`).
- **2025-07-27 effect stacking rule** — superseded by the duel_v2 effect queue (staged [[spell-system]] "Effect queue semantics").
- **2026-01-30 server docs v2** — most v2 docs are now stubs or rewritten (`server:docs/spell_system/*-v2.md` new, `docs/match/*` stubs per staged report `server-match.md`).

## Open questions

| Question | Verdict 2026-09-07 |
|---|---|
| Canonical spell set: 63 YAML vs 19 client folders? | DONE server-side: 14 YAML. Client `Resources/Spells/` still has 21 old-id folders (+`magic_arrow`, `mirror_reflection`); only those two overlap the catalog. Icon/VFX/SFX production for the other 12 is still open. |
| New roster: how many, style, default race, elements, resistances? | DONE: 6 races, default `human`, elements stored but inactive in duel_v2, `spell_resistances` all `{}` (staged `server/progression.md:24-35`). |
| Duel renders races: SpriteFrames or rig? | DONE: SpriteFrames via `RaceSpriteAnimator` (fact 1). |
| `stun/slowdown/absorb/mana_drain`: handlers or cut? | DONE: cut with the catalog (fact 2). |
| Endless Story: abandoned or rewritten without OpenAI? | OPEN. Module still registered, RPC commented, client caller still present, `create_character_match_story` reported broken in staged [[rpcs]]. Needs a human decision (delete vs. keep). |
| Race elements vs server spell schools | DONE by redesign: race combat bonuses inactive; catalog spells carry `school: neutral` (`server:data/spells/firebolt.yaml:6`). Whether elements come back is a design question, not a code drift. |
| Secrets to rotate and purge from history | OPEN, unchanged (fact 5). |
| Deploy from Mac | DONE for Android APK (`client:deploy.sh`, `client:CLAUDE.md` "Deploy to Android Phone"). |
| **New:** prod DB migration strategy | OPEN. Tracked history ends at `000016`; working tree restarts at `000001..000003` ("fresh development baseline, not an upgrade", `server:docs/spell_system/database-v2.md:3`). Pushing to `main` triggers Argo auto-sync + `migrate-custom`; behaviour of golang-migrate against a DB at version 16 with a lower-numbered set is unverified (expect failure/dirty). |
| **New:** is the k8s prod still alive / wanted? | OPEN. Chart + Argo Application present; no doc in either repo describes it any more (grep for Argo/helm/elanon.pl in `server:docs/*.md` = none); only the staged [[infra-and-deploy]] does. |
