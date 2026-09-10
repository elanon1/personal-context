---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, server, match-engine, architecture]
sources: ["server:docs/match/GUIDE-v2.md", "server:docs/match/README.md", "server:docs/match/architecture.md", "server:docs/match/phases.md", "server:docs/match/state-management.md", "server:docs/match/communication.md", "server:docs/match/matchmaking.md", "server:docs/QUICKSTART-v2.md", "server:docs/DOCUMENTATION-INDEX.md", "server:CLAUDE.md", "server:AGENTS.md"]
---

# Server architecture: Nakama plugin and match engine

Hexbane's backend is one Go plugin (`backend.so`) loaded by **Nakama 3.27.0**. It registers RPCs, auth hooks and two real-time match handlers. Combat runs ruleset `duel_v2`, catalog `duel_v2.4`, protocol 2, at **100 ms per tick** (`server:modules/spell_system/version.go:3-8`). Payload shapes for opcodes are in [[combat-v2]] and [[opcodes]]; RPCs in [[rpcs]]. Build and run workflow: [[dev-setup]].

## Plugin entry and module map

`server:modules/main.go:27-117` is `InitModule`. It loads `.env` with godotenv and **fatals if the file is missing** (`main.go:32-34`), then registers modules in this order:

| Order | Package (`server:modules/…`) | Registers | Notes |
|---|---|---|---|
| 1 | `race` | RPC `get_races`, `get_race`; loads race registry from DB | must run before `character` (`main.go:36`) |
| 2 | `character` | RPCs `create_character`, `debug_create_character`, `get_character_by_id`, `get_my_character`, `allocate_stat_points`, `get_character_details`, `tutorial`, `set_tutorial_completed`, `get_progression` | |
| 3 | `healthcheck` | RPC `healthcheck` | |
| 4 | `match/normal_match` | match handler `normal`, matchmaker-matched hook, before-RT hook `MatchmakerAdd`, RPC `decline_match` | PvP |
| 5 | `match/ai_match` | match handler `ai_duel`, RPC `create_ai_arcane_duel` | PvE |
| 6 | `spellbook` | RPCs `get_spellbook`, `learn_spell`, `get_available_spells`, `get_player_spells`, `get_starter_spells` | |
| 7 | `playstyle` | RPCs `create_playstyle`, `delete_playstyle`, `update_playstyle`, `get_playstyles` | |
| 8 | `auth` | before/after hooks for email, Google, Apple, Game Center authentication | see [[google-auth]] |
| 9 | `social` | 15 `social_*` / friend RPCs over Nakama built-ins | |
| 10 | `notifications` | RPCs `notifications_list`, `notifications_delete` | |
| 11 | `spell_system` | RPCs `get_spell_lore`, `get_entry_spells`, `get_my_spells`, `get_spell`, `get_spell_details_yaml`; loads the catalog from `/nakama/spells` (`spell_system/init.go:36`) | catalog validated at boot, boot fails on a bad file |
| 12 | `spell_system/spell_effects` | effect engine init | |
| 13 | `endless_story` | match handler `v2_create_character`, RPC `create_character_match_story` | legacy AI character-creation match, still compiled and registered (`endless_story/create_character/init.go:25-30`) |
| 14 | `spell_effect_handlers.RegisterAllHandlers()` | binds effect types to handlers | `main.go:114` |

Supporting packages without registrations: `combat` (damage math), `progression` (XP, levels, magic points, spell slots), `skills`, `common`.

Nakama's own tables (users, friends, notifications, storage) are used as-is; the plugin adds `races`, `characters`, `character_spells`, `playstyles`, `playstyle_slots`, `account_tutorials` through three golang-migrate migrations (see [[dev-setup]] and [[database]]).

## Match engine layout

```
modules/match/
  normal_match/   PvP handler: init (RegisterMatch "normal"), join, leave, loop, matchmaker, signal, terminate
  ai_match/       PvE handler: init (RegisterMatch "ai_duel" + RPC), join, leave, loop, bot, signal, terminate
  engine/
    core/         RunLoop + DispatchMessage (loop.go), BuildPlayerState (player_setup.go), RestorePresence (rejoin.go)
    state/        MatchState, PlayerState, GameState, MatchStats, MatchLog, PhaseState interface, actions.go (cast/meditate/regen)
    phase/        one package per phase: connecting, lobby, lobby_countdown, loading, game_countdown, game, gameover
    match_types/  opcode constants (op_codes.go), ClientReady, MustMarshal
```

`engine/player/` no longer exists (deleted in the working tree). Both match handlers are thin: they call the shared `core.RunLoop` and `core.BuildPlayerState`, and only differ in join/leave rules.

### Matchmaking and match creation

- **PvP**: the before-RT hook forces every `MatchmakerAdd` to `MinCount=2, MaxCount=2, Query=""` (`normal_match/matchmaker.go:43-45`). When Nakama pairs two entries, `MakeMatch` calls `nk.MatchCreate("normal", …)` (`matchmaker.go:25`). Matchmaker interval is 3 s (`server:local.yml`).
- **PvE**: RPC `create_ai_arcane_duel` creates an `ai_duel` match and returns `{match_id}` (`ai_match/init.go:34-46`).
- **Decline**: RPC `decline_match` sends match signal `decline`; the handler broadcasts opcode 8 and sets `TerminateMatch` (`normal_match/init.go:31-53`, `signal.go:22-27`).
- Both handlers return `state.TicksPerSecond` (10) as the loop rate from `MatchInit` (`normal_match/init.go:79`, `ai_match/init.go:63`).

### Join, leave, rejoin

| Event | PvP (`normal_match`) | PvE (`ai_match`) |
|---|---|---|
| Join attempt | existing participant always allowed; otherwise reject when 2 players present (`join.go:54-70`) | existing participant allowed; reject when a human is already in (`join.go:74-87`) |
| First join | `core.BuildPlayerState` loads character + learned spells from SQL; failure broadcasts opcode 9 `{reason: player_setup_failed}` and terminates (`join.go:30-41`) | same; additionally bot slots and draft are re-rolled to match the human's `SpellSlots` (`join.go:49-66`) |
| Phase start | when 2 players are present → `PhaseConnecting` (`join.go:47-49`) | when the first human is present → `PhaseConnecting` (`join.go:68`) |
| Rejoin | `core.RestorePresence`: presence replaced, no SQL reload, no phase reset; if the current phase implements `Snapshot` (only combat) the new session gets an opcode 32 snapshot immediately (`core/rejoin.go:11-27`) | same |
| Leave | only the presence whose session id matches is removed (a stale session leaving does not evict a replacement, `leave.go:23-26`). In any pre-combat phase the match is cancelled: opcode 9 `{reason: opponent_left}` to remaining players + terminate (`leave.go:34-53`, `isPreCombatPhase` `leave.go:58-69`). During combat/game-over the match keeps running without that player. | presence removed only (`leave.go:13-17`); nothing is cancelled |
| Message auth | `core.DispatchMessage` drops messages from non-participants and from a session id that is not the current presence (`core/loop.go:113-121`) | same |

### Game loop (`core.RunLoop`, `server:modules/match/engine/core/loop.go:29-95`)

1. `Tick++`; if `TerminateMatch` return nil (Nakama ends the match).
2. During **combat** incoming messages are dispatched **before** `PhaseState.Tick`; in every other phase **after** it (`loop.go:54-71`).
3. Empty-match reclaim. `hasPlayers` is `len(Presences) > 0` for both handlers (`normal_match/loop.go:14`, `ai_match/loop.go:14`; the bot never has a presence).

| Constant | Value | Meaning | Source |
|---|---|---|---|
| `TicksPerSecond` | 10 | 100 ms tick | `state/state.go:12` |
| `JoinGraceTicks` | 300 (30 s) | a match nobody has joined yet is reclaimed after this | `core/loop.go:24` |
| `EmptyMatchTerminateTicks` | 50 (5 s) | a match that had players and now has no presences is reclaimed after this | `core/loop.go:14` |

`HasHadPlayers` flips once and never back (`state/state.go:53-57`).

## Phase state machine

`phase.Phase` values (`server:modules/match/engine/phase/phase.go:14-23`): `connecting`, `lobby_picking`, `lobby_countdown`, `loading`, `game_countdown`, `combat`, `combat_end`, `stats`. `stats` is declared but never entered. Game over runs under the `combat_end` value (`phase/game/phase.go:332`). Every phase implements `PhaseState{Tick, HandleMessage}` (`state/phase_state.go:9-12`).

Opcodes are given by number only; payloads are in [[opcodes]] and [[combat-v2]].

| # | Phase | Duration / timeout (constant) | Entry | What happens | Exit → next | Opcodes |
|---|---|---|---|---|---|---|
| 1 | `connecting` (`phase/connecting/phase.go`) | **none**: waits for readiness; only the empty-match reclaim or a leave can end it | all players present | every 1 s each human gets opcode 0 (`me` private view, `enemy` public view, `combat_protocol`, `ruleset_id`, `catalog_version`, `tick_ms`) (`:76-106`). Opcode 2 must carry `combat_protocol: 2`, otherwise opcode 30 `protocol_rejected` (`:119-125`, `:135-146`). Bots are ready from the start (`:58-60`). | all ready → broadcast opcode 1, random first drafter → `lobby_picking` (`:107-113`) | 0, 2 in, 1, 30 |
| 2 | `lobby_picking` (`phase/lobby/phase.go`) | per-player turn clock `LobbyPickingDurationTicks = 35` **seconds** (`:21`; decremented once per second for the drafting player only, `:190-195`) | | turn-based draft: opcode 4 accepted only from the drafter, rejects duplicates, standard spells, full slots (`SelectSpell` `:121-157`); next drafter by `determineNextTurn` (`lobby/utils.go:34-65`: alternate while both have room, else whoever still has room). Opcode 2 with `event_name: lobby_ready` returns the draftable spellbook (opcode 70) plus current state (`:312-331`). Bot picks a random draftable spell on its turn (`:223-251`). Clock expiry auto-fills that player, hands the turn over or auto-fills the opponent too (`:197-221`). Opcode 3 every 1 s. | all slots full → `lobby_countdown` (`:242-246`, `:298-302`) | 3, 4 in, 5, 70, 2 in |
| 3 | `lobby_countdown` (`phase/lobby_countdown/phase.go`) | `LobbyCountdownDurationTicks = 15` seconds (`:16`) | | opcode 3 `{time_remaining}` every 1 s (`:42-48`) | 0 → broadcast opcode 6 → `loading` (`:50-55`) | 3, 6 |
| 4 | `loading` (`phase/loading/phase.go`) | up to 30 s | | waits for protocol-2 `game_hud_ready` from every non-bot sender; broadcasts reliable opcode 7 when ready, otherwise opcode 9 `loading_timeout` and terminates | → `game_countdown` | 7 |
| 5 | `game_countdown` (`phase/game_countdown/phase.go`) | `GameCountdownDurationTicks = 2` (`:19`): opcode 16 sent at 2, 1, 0 once per second, transition on the third second boundary (about 3 s, `:62-78`) | | opcode 2 from a player answers with opcode 10 (`me` + `enemy`, both as **private** views, `:44-60`, `:88-105`) | `TicksLeft == -1` → `combat` | 16, 2 in, 10 |
| 6 | `combat` (`phase/game/phase.go`) | `GameTime = 180` seconds (`:27`); `time_remaining = 180 − Elapsed/10` (`:205`) | `NewGamePhaseState` (`:81-83`) | see below | any player dead or time ≤ 0 (`state/state.go:146-158`) → `combat_end` with reason `defeated` / `timeout` / `draw` (`:206-243`, `:332`) | 29 in, 30, 31, 32 |
| 7 | `combat_end` = game over (`phase/gameover/phase.go`) | 1 tick | | outcome, XP, record, skill gains, SQL update, opcode 50 per presence, then `TerminateMatch` (`:133-179`) | match ends | 50, 199 in |

Phase transitions: `MatchState.TransitionTo` is documented as the single entry point (`state/state.go:79-85`) but only the join code, connecting and combat use it; lobby, lobby_countdown, loading and game_countdown assign `ms.Phase`/`ms.PhaseState` directly (see report).

### Combat phase internals (`server:modules/match/engine/phase/game/`)

Restored stat/race/skill calculations, random-stream ownership and client metadata: [[combat-stat-rules]].

- **Logical clock.** `Elapsed` counts ticks; `now()` is `Elapsed × 100 ms` from the Unix epoch, not wall time (`phase.go:84-86`). All deadlines in payloads are ceiling to the first authoritative tick of that clock (`deadlineTick`, `phase.go:374-379`). The simulator `cmd/duel-sim` drives the same `Advance` (`phase.go:138`).
- **Command intake** (`HandleMessage` `phase.go:352-372`): only opcode 29; body ≤ 1024 bytes, unknown fields and trailing JSON rejected. `Submit` (`phase.go:93-133`) validates `client_seq` (0 invalid, equal = duplicate ignored, lower = `stale_command`), kind `cast | meditate | clear_queue`, spell ownership and mana, paralysis, meditation availability. Rejections are emitted as `action_rejected` with reasons from `reasons.go:8-19`. One pending command per player; the last one wins within a tick.
- **Advance order per tick** (`phase.go:138-304`): AI every 4 ticks (`:149-151`) → `Elapsed++` → effect expiry → release every finished cast (`cast_released`, impact scheduled at `EndsAt + travel_time`, `:159-190`) → `EffectQueue.Process` with `DeferDeath` so a whole impact batch resolves before deaths commit (`:191-201`) → end check → per player: regeneration (`RegenerateAt`, profile-derived passive/active mana and passive unpoisoned HP after an 800 ms meditation ramp, `state/actions.go:39-85`), pending → queued (`action_queued` / `queue_cleared`), queued command executed once the player is neither casting nor in recovery (`:244-303`).
- **Casting** (`state/actions.go:10-37`): mana is spent at cast start; `RecoveryUntil = EndsAt + recovery_time`; starting a cast stops meditation. Paralysis interrupts a cast in flight and keeps the mana (`InterruptCastAt`, `actions.go:87-98`).
- **Impact** (`apply_spell_effect.go:11-39`): target is the opponent if any effect targets the enemy, otherwise self; a hostile spell hitting a target with `reflection` swaps caster and target and consumes the effect (`spell_reflected`), then the final target rolls dodge once; on a hit `spell_effects.ApplyEffects` runs the handlers and `spell_impact` is emitted.
- **Broadcasts** (`Tick` `phase.go:305-335`): every event is sent every tick; `action_queued`, `queue_cleared`, `action_rejected` go only to the acting player on opcode 30, everything else to all on opcode 31. Snapshots (opcode 32) go per player every 2 ticks (200 ms) and on the final tick; each contains only the recipient's `queued` command (`Snapshot` `:336-351`).
- **End** (`:206-243`): `IsOver` is true when any player is dead or `time_remaining ≤ 0`. Reason `defeated` if exactly one player is alive, `timeout` if all alive at 0 s, otherwise `draw` (simultaneous death). Effects and shields are cleared, `match_ended` emitted, `GameState.Winner/Defeated/MatchResult` set.

### Effect queue (`server:modules/spell_system/spell_effects/queue.go`)

- Owned by the combat phase (`GamePhaseState.EffectQueue`), never shared across matches.
- `Add` (`:53-81`): status effects (`poison`, `paralyze`, `shield`, `reflection`, `regeneration`, `delayed_hex`) do not stack: a second instance on the same target is dropped; paralysis during `ParalyzeImmuneUntil` is dropped. Effects start immediately when `StartAt ≤ now`.
- `Process` (`:159-230`): drains all due work in deterministic order: earliest time first, then owner order (which alternates every other 100 ms slot), then sequence id. Effects fire `start`, `tick` (every `interval`), `end` through `ApplyEffect` (`engine.go:21`). `delayed_hex` fires once at its due tick.
- Removal reasons: `expired`, `broken`, `depleted`, `consumed`, `cured`, `match_ended` (`state/match_log.go:27-33`, `queue.go:250-259`). Removing paralysis grants 3 s immunity (`queue.go:124-126`). Lifecycle events reach the phase through `Sink` (`phase.go:142-148`).
- Registered handler types: `damage`, `heal`, `poison`, `reflection`, `shield`, `paralyze`, `cure`, `delayed_hex`, `regeneration`, `dispel`, `consume_venom` (`spell_system/types.go:15-25`, `effect_handlers/registry.go`). Handler behaviour and the 14-spell catalog: [[spell-system]].

## State structs (`server:modules/match/engine/state/`)

Cleanup 2026-09-08 retains `MatchLog` and its current lifecycle. The old `CastInterruptions` buffer was removed from match/effect contexts; live interruption events are emitted by `spell_effects.ApplyEffect` through the queue sink. No opcode or payload shape changed.

**`MatchState`** (`state.go:35-59`): `MatchId`, `Presences map[userId]Presence`, `Players map[userId]*PlayerState`, `Game *GameState`, `Stats *MatchStats`, `Phase`, `PhaseState`, `MatchLog`, `EffectRemovalEvents`, `MatchMode` (`normal` | `ai`), `Tick`, `EmptyTicks`, `HasHadPlayers`, `TerminateMatch`, `Debug`. Helpers: `GetPlayer`, `GetOpponent(Id)`, `GetPlayerUserIds`, `GetPlayerSlots` (draft slots clamped to draftable spells, `:129-144`), `IsOver`, `GetNonBotsPlayers`.

**`PlayerState`** (`player_state.go:12-67`), built by `core.BuildPlayerState` (`core/player_setup.go:18-87`):

| Group | Fields | Notes |
|---|---|---|
| identity | `UserId`, `Username`, `RaceId`, `IsBot` | |
| resources | `Health/MaxHealth`, `Mana/MaxMana`, `Shield`, `IsAlive`, `DiedTimestamp` | derived from the shared combat profile for humans and bots; see [[combat-stat-rules]] |
| stats and skills | `Strength`, `Intelligence`, `Dexterity`, `SkillMeditation`, `SkillSpellResistance`, `SkillMagery`, `SkillGains` | active in combat; humans initialize a gain tracker, bots do not persist gains |
| spells | `SpellSlots` (draft slots this match, clamped to learned spells), `MaxSpellSlots` (character entitlement), `SpellBook` (learned + standard), `StandardSpells`, `SelectedSpells` (standard + drafted, castable) | standard spells are carried, never drafted (`player_setup.go:44-54`, `DraftableSpells` `player_state.go:426-436`) |
| combat | `CastingSpell`, `IsMeditating`, `Effects`, `RecoveryUntil`, `MeditationReadyAt`, `ParalyzeImmuneUntil`, private fractional regeneration carries, `DeferDeath` | deadlines on the phase's logical clock (`player_state.go:13-19`) |

`GetSnapshot` (`player_state.go:69-83`) is the `state` object inside opcode 32: `hp`, `hp_max`, `mana`, `mana_max`, `shield`, `casting`, `meditating`, `paralyzed`, `poisoned`, `effects`. `ToPrivateView` / `ToPublicView` (`player_state.go:241-266`, `player_state/types.go:8-41`) feed opcodes 0 and 10.

**`GameState`** (`game_state.go:11-20`): `TimeRemaining`, `Winner`, `Defeated`, `MatchResult` (`draw|win|lose`); `SpellQueue` and `ActiveEffects` are unused. **`MatchStats`** (`match_stats.go`) is allocated but never written.

## Concurrency model

Nakama runs all handlers of one match (`MatchLoop`, `MatchJoin`, `MatchLeave`, `MatchSignal`) on a single goroutine, so match state is single-threaded per match. The `sync.RWMutex` embedded in `MatchState` and `PlayerState` is defensive only (`state.go:27-34`). The rule that follows: no match-scoped data may be shared across matches, which is why the effect queue is phase-owned. `PlayerState` mutators (`TakeDamage`, `Heal`, `AddMana`, `AddEffect`, `TryCastSpellAt`, `RegenerateAt`, …) take the lock; `*Unsafe` variants assume it is held.

## AI match specifics (`server:modules/match/ai_match/`)

- Bot user id `"0000"`, username `Bot`, never has a presence and is never persisted (`bot.go:16`, `gameover/phase.go:201-204`).
- Race picked at random from the registry at `MatchInit` (`bot.go:53-59`, `init.go:60`); base stats are the per-race level-1 spreads from migration `000002_reference_data` (`bot.go:34-41`); HP/mana and combat traits derived through the same `combat.Profile` as humans.
- Spell slots = `progression.CalculateSpellSlots(1, traits.SpellSlotBonus)`, re-synced to the human's slot count on join; the draft is `spellbook.GetRandomSpells(n)` plus the standard spells (`bot.go:93-110`, `join.go:49-66`).
- Ready automatically in `connecting`; picks a random draftable spell on its lobby turn.
- Combat brain `runAI` (`phase/game/ai.go:19-131`) runs every 4 ticks, reads only public state, reacts to a cast or effect only after it has been visible for 400 ms, and submits ordinary opcode-29 commands. Policies `pressure` (default in live matches), `sustain`, `control` change the candidate spell order; only `cmd/duel-sim` assigns the other two (`cmd/duel-sim/main.go:88`).

## Game over and progression hook (`server:modules/match/engine/phase/gameover/phase.go`)

- Outcome from health: one alive → `victory`/`defeat` with reason `defeated`; both alive → `draw` with reason `timeout`; both dead → `draw` with reason `draw` (`:105-131`). A draw is not counted as a loss (`:234-236`).
- Per human player: `character.FindByUserId` → XP via `progression.CalculateMatchXP` (win 100 + 50 first win of the day; loss/draw 30; +50 for the first match of the day, `progression/constants.go:8-11`, `xp.go:92-110`) → `char.AddExp` handles level-ups (5 stat points per level, magic points by level band, spell-slot ladder) → wins/losses, `LastMatchDate`, `LastFirstWinDate` → `char.UpdateMatchResult` (`character/db.go:334`).
- Skill gains would be applied here but `SkillGains` is always nil in duel_v2, so the `skill_gains` block in opcode 50 reports zero change.
- Result payload (`PlayerMatchResult` `:23-87`) is sent only to players who still have a presence (`:335-350`); a disconnected player gets no opcode 50.

## Snapshot allocation (2026-09-10)

`phase/game/snapshot.go` serializes typed envelopes/player/action payloads. Wire semantics and recipient privacy are unchanged; tick/snapshot cadence stays 100/200 ms. [[2026-09-10-performance]] records before/after benchmarks and 100–10,000 resident-duel working sets, with explicit transport/DB/scheduling exclusions.

## Source of truth in code

- `server:modules/main.go` — plugin entry, module registration order, `.env` requirement
- `server:modules/match/normal_match/*.go` — PvP handler, matchmaker hooks, decline signal, pre-combat cancel
- `server:modules/match/ai_match/*.go` — PvE handler, bot construction
- `server:modules/match/engine/core/loop.go` — shared loop, reclaim timeouts, message authorisation
- `server:modules/match/engine/core/player_setup.go`, `rejoin.go` — player state from SQL, rejoin snapshot
- `server:modules/match/engine/phase/phase.go` and `phase/*/phase.go` — phase constants, durations, transitions
- `server:modules/match/engine/phase/game/{phase,ai,apply_spell_effect,reasons}.go` — combat tick, AI, impact, rejection reasons
- `server:modules/match/engine/state/{state,player_state,actions,game_state,match_log}.go` — state structs, cast/meditate/regen rules
- `server:modules/match/engine/match_types/op_codes.go` — opcode numbers
- `server:modules/spell_system/version.go` — protocol, ruleset, catalog version, tick length
- `server:modules/spell_system/spell_effects/{queue,engine}.go` — effect queue ordering and lifecycle
- `server:modules/progression/{constants,xp}.go` — XP constants used at game over
