---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, server, progression, races, stats, skills, combat]
sources: ["server:docs/progression/overview.md", "server:docs/progression/race.md", "server:docs/progression/stats.md", "server:docs/progression/skills.md", "server:docs/progression/progression.md", "server:docs/progression/combat.md", "server:docs/progression/match-integration.md", "server:docs/progression/modifiers.md", "server:docs/superpowers/specs/2026-09-02-race-system-redesign-design.md", "server:docs/superpowers/plans/2026-09-02-race-system-redesign.md", "client:docs/Server/progression/overview.md", "client:docs/Server/progression/race.md", "client:docs/Server/progression/stats.md", "client:docs/Server/progression/skills.md", "client:docs/Server/progression/progression.md", "client:docs/Server/progression/combat.md", "client:docs/Server/progression/match-integration.md", "client:docs/Server/progression/races_seed.sql"]
---

# Character progression and combat rules (duel_v2)

Characters keep race, STR/INT/DEX, three skills, XP/levels, stat points, magic points (MP) and draft slots. In the current prototype ruleset `duel_v2` **none of the stored stats, skills or racial combat traits change a duel**: every character fights with 200 HP and 100 mana and fixed catalog values (see [[spell-system]]). Only Human's extra draft slot is an active racial effect. `get_character_details` reports `stat_bonuses_active: false`, an empty `modifiers` array and only the slot trait (`server:modules/character/details.go:214,306-320`).

## Races

Six races seeded by migration `000002_reference_data` (`server:db/migrations/000002_reference_data.up.sql`), loaded once into an in-memory registry at startup (`server:modules/race/init.go:11-18`, `registry.go`). `race.DefaultRaceId = "human"` (`server:modules/race/types.go:6`). Limits apply to **effective** stats (base + modifier); `0` means no limit.

| Race | `race_id` | STR/INT/DEX modifier | min STR | max STR | min INT | max INT | min DEX | max DEX | `traits` |
|---|---|---|---:|---:|---:|---:|---:|---:|---|
| Human | `human` | +20/+20/+20 | — | 250 | — | 250 | — | 250 | `{"spell_slot_bonus": 1}` |
| Elf | `elf` | 0/+70/+30 | — | 160 | 200 | — | — | — | `{"mana_regen_multiplier": 1.4}` |
| Dark Elf | `dark_elf` | 0/+100/0 | — | 120 | 220 | — | — | 140 | `{}` |
| Shadow | `shadow` | 0/0/+100 | — | 150 | — | 180 | 200 | — | `{"dodge_per_dex": 0.05, "dodge_cap": 25.0}` |
| Gnome | `gnome` | 0/+50/+50 | — | 110 | 150 | — | 150 | — | `{"mana_cost_multiplier": 0.75}` |
| Orc | `orc` | +110/0/0 | 250 | — | — | 120 | — | 130 | `{"damage_stat": "strength", "health_multiplier": 1.15, "paralyze_duration_multiplier": 0.5}` |

Also stored per race (character-creation and menu metadata only, no duel effect): `casting_time_modifier` (Shadow −2, Gnome −6, Orc +3, others 0), `primary_element`/`secondary_element` with bonuses (Human neutral +20; Elf water +25 / ice +12; Dark Elf toxic +40 / mind +20; Shadow air +25 / mind +12; Gnome lightning +25 / fire +12; Orc earth +25 / fire +12), `spell_resistances` (all `{}`).

Traits are decoded into a typed `race.Traits` struct; an unknown key or wrong type fails startup (`server:modules/race/traits.go:54-119`). Every trait is returned by `get_races`/`get_race` with neutral defaults for missing keys (`mana_regen_multiplier`, `mana_cost_multiplier`, `health_multiplier`, `paralyze_duration_multiplier` = 1; dodge overrides = 0; `damage_stat` = "").

**Active trait:** only `spell_slot_bonus`, read in `server:modules/character/character.go:75,266` and `details.go:210,316`. The other seven keys have no consumer outside `modules/combat` helper signatures and tests (verified by grep on 2026-09-07). Do not present them as active bonuses.

## Stats

- Creation distributes exactly `CreationPoints = 400` base points over STR/INT/DEX (`server:modules/progression/constants.go:17`, `server:modules/character/validate.go:31-34`).
- Checks at creation (`validate.go:28-71`): total = 400; race exists; **effective** stat ≥ `MinStatValue = 10`; race floors/ceilings on effective stats; base ≥ 1. Name 3–20 characters.
- Defaults when unspecified (debug RPC only): 133/134/133 (`constants.go:19-21`).
- Effective = base + race modifier, floored at 1 (`server:modules/combat/stats.go:107-124`). Both base and effective columns are persisted.
- Level-up points: `allocate_stat_points` accepts any non-negative subset of unspent points (the "total must equal unspent" check is commented out, `server:modules/character/rpc.go:223-235`). Each stat is validated against race limits **before** mutation (`character.go:140-177`).
- `get_character_details.stats` returns `base`, `racial`, `effective`, `min`, `max` per stat (`details.go:71-85`).

Seed distributions used by `make db-seed` (legal for every race): Human 133/134/133, Elf 120/200/80, Dark Elf 110/180/110, Shadow 120/150/130, Gnome 100/160/140, Orc 180/110/110 (`server:scripts/seed_dev_accounts.sh:64-69`).

## Skills

**Direction agreed 2026-09-08:** resistance, regeneration and skills must return. Their implementation and tests are retained during cleanup; activation scope is awaiting clarification. The following describes the still-active fixed-value ruleset, not the intended final combat design.

Meditation, Spell Resistance and Magery are stored as 0–100 floats (`skill_meditation`, `skill_spell_resistance`, `skill_magery`). They are shown in details with `tier = ceil(value/10)` and groups `core`/`defense` (`server:modules/character/details.go:290-304`).

In `duel_v2` they do nothing and never gain: `BuildPlayerState` sets `SkillGains: nil` (`server:modules/match/engine/core/player_setup.go:77`), the damage path uses `TakeDamageWithEvents` (no trigger), and the game-over phase only applies gains when the tracker is non-nil (`server:modules/match/engine/phase/gameover/phase.go:239-255`). `modules/skills` (random gain chances 15/8/3/0.5 % and `+0.1`) remains as dormant helpers.

## XP and levels

`server:modules/progression/constants.go`, `xp.go`.

| Constant | Value |
|---|---:|
| `MaxLevel` | 30 |
| `XPWin` | 100 |
| `XPLoss` | 30 (also awarded for a draw) |
| `XPFirstWin` | +50, winners only, once per calendar day (`last_first_win_date`) |
| `XPDaily` | +50, first match of the day for any outcome (`last_match_date`) |
| `StatPointsPerLevel` | 5 |

XP required to *reach* a level: `XPForLevel(1)=0`, `XPForLevel(2)=100`, `XPForLevel(n)=int(100·1.5^(n−2))` (`xp.go:10-18`). Experience is cumulative and never reset. Computed from the formula:

| Level | Total XP |
|---:|---:|
| 2 | 100 |
| 3 | 150 |
| 4 | 225 |
| 5 | 337 |
| 8 | 1 139 |
| 10 | 2 562 |
| 12 | 5 766 |
| 15 | 19 461 |
| 20 | 147 789 |
| 25 | 1 122 274 |
| 30 | 8 522 269 |

## Magic points

MP are earned on level-up and spent on learning spells (`server:modules/progression/magic_points.go`). `GetMagicPointsForLevel(1) = 0`; per level reached: 2–5 → 2, 6–10 → 4, 11–20 → 10, 21–30 → 15.

| Level reached | Cumulative MP |
|---:|---:|
| 5 | 8 |
| 10 | 28 |
| 15 | 78 |
| 20 | 128 |
| 25 | 203 |
| 30 | 278 |

Available MP = `magic_points − magic_points_spent` (CHECK-constrained ≥ 0). Every selectable spell costs its explicit `magic_point_cost` (5 for all 12 in the catalog); standards cost 0 and are rejected by `learn_spell`. Collection size is **not** capped by draft slots (`server:modules/spellbook/rpc.go:156-171`). A learn is one transaction: insert ownership row, then debit MP with a guard against overspending (`server:modules/character/db.go:248-293`).

## Draft slots

`server:modules/progression/constants.go:26-33`, `spell_slots.go`. Slots limit how many owned spells a player drafts into a match; the two standard spells are carried on top.

| Level | Slots (other races) | Slots (Human, `spell_slot_bonus` 1) |
|---:|---:|---:|
| 1 | 3 | 4 |
| 4 | 4 | 5 |
| 8 | 5 | 6 |
| 12 | 6 (cap) | 7 (cap) |

`CalculateSpellSlots(level, bonus) = min(3 + bonus + unlocks, 6 + bonus)`. `characters.spell_slots` is persisted at creation (`character.go:105`) and rewritten on level-up (`character.go:117-135`); the column CHECK is 3–7. Details report `spell_slots_unlocked` (current) and `spell_slots_max` (6 + bonus) (`details.go:370-376`).

## Combat rules (duel_v2)

Real-time 1v1, no movement, logical clock of 100 ms ticks, 180 s safety limit (`server:modules/match/engine/phase/game/phase.go:27`). Both players start at **200 HP / 100 mana** (`player_setup.go:33-34`); the duel simulator uses the same constants.

### Actions

- Commands: `cast` (with `spell_id`), `meditate`, `clear_queue` (`phase.go:110`). Rejection reasons: `not_participant`, `combat_ended`, `invalid_client_seq`, `stale_command`, `unknown_command`, `unknown_spell`, `insufficient_mana`, `unexpected_spell_id`, `paralyzed`, `meditation_unavailable` (`reasons.go`); at start: `insufficient_mana`, `busy`, `paralyzed`, `dead`, `invalid_spell` (`phase.go:381-394`).
- Mana is charged once when the cast starts and never refunded (`server:modules/match/engine/state/actions.go:29`). Casting ends automatically at `now + casting_time`; recovery blocks the next action until `cast end + recovery_time` (`actions.go:30-32`).
- One private queued action per player; a new command replaces it; `clear_queue` removes it. The queued action starts once the player is idle (not casting, past recovery, not paralysed) (`phase.go:244-303`).
- Only paralysis interrupts a cast (`paralyze.go:39-62`, `actions.go:87-98`): the cast is cancelled, mana stays spent, recovery restarts from the interruption. Damage never interrupts casting or meditation.
- Impact time = cast end + `travel_time` (0 for the whole catalog). Target is the enemy if any effect targets `enemy`, otherwise self (`phase.go:170-180`).

### Tick order (`phase.go:138-304`)

1. Bots decide every 4th tick; `Elapsed++`.
2. Expire statuses (hex excluded).
3. Release every completed cast of both players (ids sorted) and schedule impacts, before any impact can interrupt.
4. Process due impacts and pulses in deterministic order (time, then seat with priority alternating by tick parity, then sequence); deaths are deferred until the whole batch resolves.
5. Commit deaths. If someone is dead or time is up: `defeated` (one survivor), `draw` (both dead), `timeout` (both alive at 0 s). All effects, impacts and queues are cleared; `match_ended` is emitted.
6. Regenerate mana; then queue handling and action starts for living, non-paralysed players.

Simultaneous lethal impacts are a draw. Timeout is a draw regardless of HP.

### Resources

- Passive mana: 1 per second, integer carry, also under poison and paralysis (`actions.go:62-64`). No passive HP regeneration.
- Meditation: refused while poisoned, paralysed, casting, in recovery or at full mana (`actions.go:42`). 0.8 s warm-up, then an additional 10 mana/s with millisecond carry (`actions.go:49,69-77`). Stops on a successful cast, poison, paralysis, or full mana (`actions.go:33,65-67,79-84`).
- Mana is capped at 100 and HP at 200 (`player_state.go:186-191,208-211`).

### Statuses and counters

One instance per kind per target; re-application does nothing (no refresh, no stack, no refund). Details and reasons in [[spell-system]].

| Effect | Rule |
|---|---|
| Poison | 6 s, 2 damage at seconds 1–5 (10 total). Stops meditation immediately, blocks starting it and blocks regeneration pulses; direct heals work. Passive mana continues. |
| Paralysis | 1 s. Blocks all actions, clears the queue, interrupts casting and meditation. Then 3 s immunity. Damage does not remove it. |
| Mirror (reflection) | One charge for 3 s. Reflects a whole hostile package once at impact; the reflected package cannot consume a second mirror. Self-target spells never reflect. |
| Barrier (shield) | Absorbs up to 22 damage for 3 s; removed as `depleted` when damage exceeds it; hostile statuses still apply. |
| Delayed Hex | Visible marker, 28 damage after 2.5 s. Application reflects, detonation does not. Cleanse removes it; Barrier absorbs the detonation. |
| Regeneration | 5 HP at seconds 1–5 (25 total); pulses under poison are lost. |
| Cleanse | Removes hex first, else poison. Never removes paralysis or immunity. |
| Dispel | Removes mirror, else barrier, else regeneration from the enemy; is itself reflectable. |
| Consume Venom | If the final target carries poison owned by the final caster: remove it and deal 18; otherwise nothing. |

## Match integration

- `core.BuildPlayerState` loads the character and owned spells, sets 200/100, clamps draftable slots to the number of owned spells while exposing the full entitlement as `max_spell_slots`, and injects both standard spells into `SelectedSpells`/`SpellBook` (`server:modules/match/engine/core/player_setup.go:18-87`).
- Lobby draft offers `DraftableSpells()` (owned minus standards), rejects standard ids and auto-picks from the same pool (`server:modules/match/engine/phase/lobby/phase.go:130,161,226,275`).
- A bot copies the human's `SpellSlots`/`MaxSpellSlots`, drafts that many random non-standard spells and carries both standards (`server:modules/match/ai_match/join.go:49-66`, `server:modules/spellbook/db.go:70-78`).
- Rejoin keeps the existing player state; no reload, no HP/mana reset (`server:modules/match/engine/core/rejoin.go`). A combat snapshot restores the client ([[combat-v2]]).
- Game over (`server:modules/match/engine/phase/gameover/phase.go:133-332`): per human player: XP via `CalculateMatchXP`, `char.AddExp` (level, stat points, MP, slots with race bonus), `last_match_date = now`, wins++ on victory (+`last_first_win_date` if first win today), losses++ only on `defeat`; draws change neither counter. Persisted with `UpdateMatchResult`. Bots are not persisted. The per-player result message (opcode 50) carries XP, level-up, skill before/after (always equal), stats and record.

## Known code smells (not rules)

- `get_progression` now uses `XPToNextLevel` (clamped to zero) and `GetNextSpellSlotLevel`, matching the active `{4,8,12}` ladder. Fixed 2026-09-08; regression tests cover unlock boundaries, level cap and stale-level XP.
- `NewCharacter` comments still describe an empty roster / `DefaultRaceId == ""` (`character.go:67-68`).
- `modules/combat` (damage/resistance/dodge/regen formulas, passive HP regen constants) and `modules/skills` gain helpers are dormant; `CalculateEffectiveStats` is the only live call.

## Source of truth in code

- `server:db/migrations/000002_reference_data.up.sql` — race rows
- `server:modules/race/types.go`, `traits.go`, `db.go`, `registry.go` — race model, trait parsing, registry
- `server:modules/progression/constants.go`, `xp.go`, `magic_points.go`, `spell_slots.go` — XP curve, rewards, MP, slot ladder
- `server:modules/character/validate.go`, `character.go`, `details.go`, `rpc.go` — creation rules, allocation, level-up, details payload
- `server:modules/match/engine/phase/game/phase.go`, `apply_spell_effect.go`, `reasons.go` — tick loop, reflection, rejection reasons
- `server:modules/match/engine/state/actions.go`, `player_state.go` — cast/meditate/regen/damage rules
- `server:modules/match/engine/core/player_setup.go`, `server:modules/match/engine/phase/gameover/phase.go` — match entry and rewards
- `client:Core/Characters/StatAllocation.cs`, `client:Core/Characters/RaceCatalog.cs` — client mirrors of creation constants and race presentation
