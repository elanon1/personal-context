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

# Report: server match engine + dev workflow (area `server`)

Staged outputs:
- `staging/server/architecture.md`
- `staging/server/dev-setup.md`
- `staging/plans/server-devlog-summary.md`

All claims verified against the **working tree** of `/Users/elanon/GolandProjects/hexbane-server` (branch main, uncommitted duel_v2 redesign). Read-only; nothing in either repo was touched.

## 1. Input → verdict

| Input | Verdict |
|---|---|
| `server:docs/match/GUIDE-v2.md` | merged into `architecture.md` (module map, loop, termination, thread-safety, game-over flow kept after correction); **fully superseded** — every combat section, opcode table, timings and phase durations are wrong (see §2) |
| `server:docs/match/README.md` | dropped: stale (pre-`engine/` layout, opcodes 11–22, 300 s combat, 50-tick termination); the two true facts (phase list, 2-player matchmaking) are in `architecture.md` |
| `server:docs/match/architecture.md` | dropped: stale/duplicate of GUIDE-v2 with older paths (`modules/match/types.go`, `state/`), generic filler ("Security architecture", "Scalability") with no code behind it |
| `server:docs/match/phases.md` | dropped: stale; phase list merged as the corrected table in `architecture.md`; claimed `ChangePhase` in `modules/match/utils.go` does not exist (`TransitionTo` in `state.go:81`) |
| `server:docs/match/state-management.md` | dropped: stale (old `PlayerState` shape, `HaveMana` inverted, `Effects` keyed by string, `LastCastTick`, join code in `modules/match/join.go`); struct tables rewritten from code in `architecture.md` |
| `server:docs/match/communication.md` | dropped: stale; opcodes 11–14, 21–22 and payloads are retired; opcode numbering covered by the `protocol` agent, phase↔opcode mapping kept by number in `architecture.md` |
| `server:docs/match/matchmaking.md` (not assigned, skimmed) | dropped: stale path (`modules/match/matchmaker.go`) and `Query = "*"` (code sets `""`, `normal_match/matchmaker.go:45`); the true parts are 4 lines in `architecture.md` |
| `server:docs/match/api-reference.md` (not assigned) | not merged; only the superseded banner was checked. Belongs to the `rpcs`/`protocol` agents. |
| `server:docs/QUICKSTART-v2.md` | merged into `dev-setup.md` (workflow skeleton) after correction: Go 1.21+ → 1.24.3, missing `.env` requirement, log messages, `LOG_LEVEL` env var (does not exist), file paths (`cast_spell.go`, `meditate.go`, `modules/match/engine/player/`, root `main.go`), "Loaded 10 spells / 3 races", `characters_test.go` example, `utils.ChangePhase`, `hexbane-server/` import path (module is `hexbane`) |
| `server:docs/DOCUMENTATION-INDEX.md` | dropped: not documentation (index of files that are now stubs or stale); its "current" markers are wrong for progression docs (races listed as Human/Elf/Dark Elf) |
| `server:docs/devlog.md` | merged into `plans/server-devlog-summary.md` as a dated English timeline; not copied |
| `server:CLAUDE.md` | merged (module list, spell dir, migrations) with corrections: "10 ticks/second" is true but the doc's phase list omits nothing; `endless_story` described as "AI-driven character creation with OpenAI" is dead (no OpenAI code), effect-type list is stale (no Stun/Slowdown/Absorb/ManaDrain; missing cure/delayed_hex/regeneration/dispel/consume_venom) |
| `server:AGENTS.md` | merged (Go version, make targets, conventions); stale: "Spell assets live in `data/spells/<school>/<tier>/`" (now flat `data/spells/*.yaml`) |
| `server:Makefile` | merged into `dev-setup.md` (all targets verified) |
| `server:docker-compose.yml`, `docker-compose.debug.yml`, `docker-compose.prod.yml` | merged |
| `server:Dockerfile`, `Dockerfile.debug` | merged |
| `server:local.yml` | merged |
| `server:.env.dist` | merged, names only |
| `server:scripts/*` | merged (each script summarised) |
| `server:cmd/duel-sim/main.go` | merged (flags, purpose) |
| `server:rebuild.sh` | merged (marked Linux-only legacy flow) |

**Fully superseded files** (safe to delete from the server repo once the vault holds the staged docs): `docs/match/GUIDE-v2.md`, `docs/match/README.md`, `docs/match/architecture.md`, `docs/match/phases.md`, `docs/match/state-management.md`, `docs/match/communication.md`, `docs/match/matchmaking.md`, `docs/QUICKSTART-v2.md`, `docs/DOCUMENTATION-INDEX.md`. `docs/match/api-reference.md` and `docs/API-REFERENCE-v2.md` were not assessed here.

## 2. Discrepancies found (doc said X, code says Y)

### Timings and constants
| Doc claim | Code |
|---|---|
| GUIDE-v2: Lobby picking "60 ticks (6 seconds)" | per-player turn clock of 35 **seconds**, `LobbyPickingDurationTicks = 35`, decremented once per second for the drafting player only — `phase/lobby/phase.go:21`, `:190-195` |
| GUIDE-v2: Lobby countdown "3 ticks (300ms)" | 15 seconds, `LobbyCountdownDurationTicks = 15`, decremented once per second — `phase/lobby_countdown/phase.go:16`, `:42-48` |
| GUIDE-v2: Game countdown "2 ticks (200ms)" | `GameCountdownDurationTicks = 2` but stepped once per second: broadcasts 2, 1, 0 and transitions on the third second (≈3 s) — `phase/game_countdown/phase.go:19`, `:62-78` |
| GUIDE-v2: Combat "300 seconds (3000 ticks)" and diagram "300 ticks = 30 seconds"; README: "Time limit of 300 seconds" | `GameTime = 180` seconds — `phase/game/phase.go:27`; `time_remaining = 180 − Elapsed/10` (`:205`) |
| GUIDE-v2: "Tick Rate: 30 ticks/second (`state.TicksPerSecond`)" (same doc says 10 elsewhere) | `TicksPerSecond = 10` — `state/state.go:12` |
| README: "Automatic termination after 50 empty ticks"; state-management: `EmptyTicks > 50` | 50 ticks (5 s) only after the match has had players; 300 ticks (30 s) before the first join — `core/loop.go:14`, `:24`, `:79-91` |
| GUIDE-v2 / README: "Player data sync every 200ms when changes detected", snapshot comparison | snapshots (opcode 32) are sent unconditionally every 2 ticks and on the final tick; no change detection — `phase/game/phase.go:321-329`. `PlayerSnapshot.IsEqual` still exists but is unused by the phase (`player_state/snapshot.go:17`) |
| GUIDE-v2: "AI every 5 ticks (500ms)" | every 4 ticks (`g.Elapsed%4 == 0`) — `phase/game/phase.go:149`; reactive decisions wait 400 ms — `ai.go:18`, `:38`, `:54` |
| GUIDE-v2: "Bot pre-created with 4 random spells" | bot drafts `CalculateSpellSlots(1, bonus)` random spells plus the standard spells, re-rolled to the human's slot count on join — `ai_match/bot.go:93-110`, `join.go:49-66` |
| GUIDE-v2: bot "picks random spell from SelectedSpells, no strategic decision making" | policy-driven AI (`pressure`/`sustain`/`control`) with cleanse/heal/mirror/paralysis/dispel reactions — `phase/game/ai.go` |
| GUIDE-v2: meditation "random 10–30 mana + skill/DEX bonuses" | deterministic: 1 mana/s base, +10 mana/s while meditating after an 800 ms ramp; poison/paralyze stop it; full mana stops it — `state/actions.go:39-85` |
| GUIDE-v2 / CLAUDE.md: "MaxHealth = STR, MaxMana = INT", casting-time/dodge/regen modifiers from DEX, race resistances | fixed 200 HP / 100 mana; no stat-derived combat modifiers; `PlayerState` has no `CastingTimeModifier`, `DodgeChance`, `ManaRegenBonus`, `SpellResistances` fields — `core/player_setup.go:33-34`, `state/player_state.go:12-67`. Stats and skills are loaded but unused in duel_v2 |
| GUIDE-v2: skill gains applied at game over; "Skill progression tracking" | `SkillGains` is set to `nil` for every player (`core/player_setup.go:77`), so nothing is ever gained; game-over code path is dead for skills (`gameover/phase.go:240`) |
| GUIDE-v2: winner = "higher remaining health" | winner = the only player with `Health > 0`; both alive at timeout → draw with reason `timeout`; both dead → draw — `gameover/phase.go:105-131` |
| GUIDE-v2: "Loss: 30 XP … Daily bonus +50" (listed as win-only in one place) | daily +50 applies to wins and losses, first-win +50 to wins only — `progression/xp.go:92-110` |

### Opcodes and messages (numbers only; payloads are the `protocol` agent's area)
| Doc claim | Code |
|---|---|
| GUIDE-v2 table: `OpGameCountdown = 9` | 16 (`match_types/op_codes.go:24`); 9 is `OP_MATCH_CANCELED` |
| README/GUIDE/communication: combat opcodes 11, 12, 13, 14, 15, 21–28 | retired; combat uses 29 (in), 30, 31, 32 (out) — `op_codes.go:34-39`. Files `cast_spell.go`, `meditate.go`, `release_spell.go`, `timer_update.go`, `update_players.go`, `types.go` under `phase/game/` are deleted |
| GUIDE-v2 / README: opcode 8 and 9 not listed | 8 `OP_MATCH_DECLINED` (decline signal), 9 `OP_MATCH_CANCELED` (`player_setup_failed`, `opponent_left`) — `normal_match/signal.go:24`, `join.go:36`, `leave.go:37-50` |
| communication.md: `OP_CLIENT_READY` payload `{user_id}` | connecting phase requires `combat_protocol: 2` and ignores any user id in the payload (sender is the presence) — `phase/connecting/phase.go:119-146`; opcode 30 `protocol_rejected` on mismatch |
| GUIDE-v2 / phases.md: `MatchLog` is broadcast as opcode 13 every tick | `MatchLog` is appended by effect handlers and **discarded** each tick without being sent (`phase/game/phase.go:204`, `:330`) |
| phases.md: "Combat End" and "Stats" phases "to be implemented" | `combat_end` is the game-over phase (`phase/game/phase.go:332`); `PhaseStats` is an unused constant |

### Paths and structure
| Doc claim | Code |
|---|---|
| README/architecture/phases/state-management/matchmaking: `modules/match/{init,types,loop,join,leave,matchmaker,signal,utils}.go`, `modules/match/state/`, `modules/match/phase/`, `modules/match/player/` | `modules/match/normal_match/`, `ai_match/`, `engine/{core,state,phase,match_types}/`; no `utils.go`, no `player/` package (deleted in the working tree) |
| GUIDE-v2 / QUICKSTART: `engine/player/player.go`, `game/cast_spell.go`, `game/meditate.go`, `utils.ChangePhase` | none exist |
| QUICKSTART: prerequisites "Go 1.21+", "PostgreSQL 12+" | Go 1.24.3 (`go.mod:3`, CI), Postgres 17.6-alpine (compose) |
| QUICKSTART: `docker logs hexbane-nakama-1`, "Loaded 10 spells", "Loaded 3 races" | container name depends on the directory; 14 spells, 6 races (`race/init.go:18`, `data/spells/`) |
| QUICKSTART: `LOG_LEVEL` environment variable | not used; `logger.level` is in `local.yml`, the debug compose passes `--logger.level DEBUG` |
| QUICKSTART: spell YAML example with `level: "1"`, `school: Arcane`, `type: attack`, `level_requirement`, effect `delay` | current catalog schema is decoded with `KnownFields(true)` (`registry.go:118`); field set is the `spells` agent's area, but the example will not load as written (unverified field-by-field) |
| QUICKSTART: import path `hexbane-server/modules/...` | module is `hexbane` (`go.mod:1`) |
| AGENTS.md: `data/spells/<school>/<tier>/` | flat `data/spells/*.yaml` (14 files); the tree layout is deleted in the working tree |
| CLAUDE.md: effect types "Damage, Heal, Poison, Reflection, Shield, Stun, Slowdown, Absorb, ManaDrain, Paralyze" | `damage, heal, poison, reflection, shield, paralyze, cure, delayed_hex, regeneration, dispel, consume_venom` — `spell_system/types.go:15-25` |
| CLAUDE.md: `endless_story/` "AI-driven character creation with OpenAI" | package contains only a `v2_create_character` match + `create_character_match_story` RPC; no OpenAI code anywhere (`grep -ri openai modules` is empty) |
| CLAUDE.md / QUICKSTART: `make migrate-down` "Rollback last migration" | true (`down 1`); `make help` additionally advertises `make debug-up`, which has no target |
| GUIDE-v2: `MatchState` thread safety "RLock for reads, Lock for writes (phase transitions, player add/remove)" | Nakama serialises handlers per match; the mutexes are documented in code as defensive only, and phase transitions/join take no lock — `state/state.go:27-34`, `:81-85` |
| GUIDE-v2: AI match "Uses `GetNonBotsPlayers()` for empty match detection"; `core/loop.go:27-28` comment says the same | both handlers pass `len(ms.Presences) > 0` — `ai_match/loop.go:14`, `normal_match/loop.go:14` (equivalent in practice, bot has no presence) |
| matchmaking.md: `Query = "*"` | `Query = ""` — `normal_match/matchmaker.go:45` |
| DOCUMENTATION-INDEX: race docs "Human, Elf, Dark Elf" marked current | six races in `db/migrations/000002_reference_data.up.sql` |

## 3. Code smells / dead code noticed

- `lobby`, `lobby_countdown`, `loading`, `game_countdown` and `lobby/debug.go` assign `ms.Phase`/`ms.PhaseState` directly, bypassing `TransitionTo` (which resets `EmptyTicks`) — `lobby/phase.go:214,244,300`, `lobby_countdown/phase.go:52`, `loading/phase.go:28`, `game_countdown/phase.go:72`. Harmless today because `EmptyTicks` is reset while presences exist, but contradicts the comment at `state/state.go:79-80`.
- `game_countdown/phase.go:88-99` and `lobby/phase.go:312-331` trust `user_id` from the opcode-2 payload to pick the presence, contradicting the rule in `core/loop.go:97-100` ("never from a user id embedded in the payload"). A client can request another participant's private game data (opcode 10) by sending their id.
- `game_countdown.GetGameData` sends the **enemy as a private view** (todo comment at `:51`), leaking the opponent's drafted spell list before combat.
- `MatchLog` entries written by every effect handler (`effect_handlers/*.go`) are discarded each tick; `MatchStats` is allocated and never written; `GameState.SpellQueue/ActiveEffects` unused; `PhaseStats` unused; `PlayerSnapshot.IsEqual`, `TakeDamageWithSkillGain*`, `TriggerMageryGain`, `ApplySkillGains` are unreachable with `SkillGains == nil`; `state.TicksPerSecond{Double,Triple,Quad,Quint,Half}` unused; `phase/game/utils.go` `DebugPlayerEffects` unused; `lobby/debug.go` only referenced from a commented-out call.
- Stats (`Strength/Intelligence/Dexterity`), skills and race modifiers are loaded into `PlayerState` and the bot but no combat code reads them (by design of duel_v2; the docs and `modules/combat` still describe scaling).
- `modules/endless_story/create_character` registers match `v2_create_character` while its RPC calls `nk.MatchCreate("create_character", …)` (`init.go:25` vs `:34`) — the RPC can never create the match it is meant for. The whole package looks like a leftover of the 2025-08 experiment.
- `.env.dist` marks `OPENAI_API_KEY` as "Required" and lists `OPENAI_*` and `PLUGIN_PATH`; no Go code reads them, yet `main.go:32` fatals if `.env` is absent. The Dockerfile works around it by copying `.env.dist` to `/nakama/.env`.
- `docker-compose*.yml` publish port 8080 on the postgres container; nothing listens.
- `Makefile` help advertises `make debug-up` (missing); `rebuild.sh` is Linux/gvm-only and duplicates `make dev`.
- `local.yml` sets `logger.level: info`, so `ms.Debug`-gated `logger.Debug` output never shows without the debug compose file.
- `lobby/phase.go:63-85` and other payload builders `panic` on marshal failure instead of using `match_types.MustMarshal`.
- `PlayerState.AddSpellToSpellbook` (`player_state.go:268-277`) appends without dedup and without a lock; the lobby relies on its own duplicate check.
- Working tree still has `git status` deletions for the entire old `bkp/`, school/tier spell tree and migrations 000001–000016; the squash to three migrations means existing dev databases must be `make db-reset` (no migration path from the old chain).

## 4. Open questions (need a human)

1. Is `modules/endless_story` (match `v2_create_character`, RPC `create_character_match_story`) still meant to ship, or should it be deleted together with the `OPENAI_*` variables and the `.env` requirement? The client uses a local tutorial now.
2. Should the opponent's spell list in opcode 10 (`enemy` private view) be treated as intended (open information after the draft) or as a leak to fix?
3. Payload-trusted `user_id` in opcodes 2 (lobby_ready, game_countdown) — bug to fix, or accepted for a two-player game?
4. Is the `MatchLog` (opcode 13 in the old protocol) permanently replaced by opcode 31 events, so the handler writes can be removed?
5. Should stats/skills remain loaded into `PlayerState` for a future ruleset, or be dropped from match state until scaling returns? Affects how much of `modules/combat`, `modules/skills` docs to keep.
6. Node version floor for `scripts/test_combat_runtime.mjs` (built-in `WebSocket`) is not pinned anywhere; the runbook says "Node with built-in WebSocket" (unverified which minimum).
7. `PLUGIN_PATH` in `.env.dist`: is anything outside the repo (Synology deployment?) using it?
