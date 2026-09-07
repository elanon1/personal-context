---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, spells, effects, balance]
sources: ["server:docs/spell_system/GUIDE-v2.md", "server:docs/spell_system/spells.md", "server:docs/spell_system/balance-v2.md", "server:docs/spell_system/balance-v2.json", "server:docs/spell_system/verification-v2.md", "server:docs/superpowers/specs/2026-09-05-spell-system-redesign.md", "server:docs/superpowers/plans/2026-09-05-spell-system-redesign.md", "client:docs/Plans/2026-09-04-standard-spells-6-slot-draft.md"]
---

# Spell system (catalog duel_v2.2)

`data/spells/*.yaml` is the only spell definition source. The whole directory is loaded atomically at plugin start; any invalid file fails startup. PostgreSQL stores ownership and saved loadouts only (see [[database]]). Combat rules that consume these definitions are in [[progression]] (section "Combat rules"); the wire protocol is in [[combat-v2]].

Version constants (`server:modules/spell_system/version.go:3-8`):

| Constant | Value |
|---|---|
| `CombatProtocol` | 2 |
| `RulesetID` | `duel_v2` |
| `CatalogVersion` | `duel_v2.2` |
| `TickMillis` | 100 |

Every spell RPC response, character details and the tutorial RPC embed `combat_protocol`, `ruleset_id`, `catalog_version` (`server:modules/spell_system/rpc.go:159-167`).

## Catalog

All 14 definitions verified against the YAML files on 2026-09-07. Every spell has `school: neutral`, `level_requirement: 1`, `travel_time: 0`. Times are seconds.

| ID | Name | Nature | Mana | Cast | Recovery | MP cost | Flags | Effect (type, value, timing, target) |
|---|---|---|---:|---:|---:|---:|---|---|
| `magic_arrow` | Magic Arrow | arcana | 3 | 0.6 | 0.3 | 0 | standard | damage 4, enemy |
| `mirror_reflection` | Mirror Reflection | arcana | 9 | 0.5 | 0.4 | 0 | standard | reflection, 3 s, self |
| `firebolt` | Firebolt | ember | 9 | 1.0 | 0.4 | 5 | starter | damage 16, enemy |
| `heavy_bolt` | Heavy Bolt | ember | 18 | 1.9 | 0.5 | 5 | starter | damage 32, enemy |
| `poison` | Poison | venom | 12 | 1.0 | 0.4 | 5 | starter | poison 2/pulse, duration 6, interval 1, enemy |
| `cleanse` | Cleanse | vitality | 6 | 0.9 | 0.3 | 5 | starter | cure, self |
| `mend` | Mend | vitality | 10 | 0.8 | 0.4 | 5 | starter | heal 14, self |
| `barrier` | Barrier | arcana | 12 | 0.9 | 0.4 | 5 | starter | shield 22, 3 s, self |
| `delayed_hex` | Delayed Hex | hex | 16 | 1.2 | 0.4 | 5 | — | delayed_hex 28, delay 2.5, enemy |
| `paralysis` | Paralysis | hex | 18 | 1.0 | 0.4 | 5 | — | paralyze, 1 s, enemy |
| `greater_heal` | Greater Heal | vitality | 19 | 1.8 | 0.5 | 5 | — | heal 32, self |
| `regeneration` | Regeneration | vitality | 13 | 1.1 | 0.4 | 5 | — | regeneration 5/pulse, duration 6, interval 1, self |
| `dispel` | Dispel | arcana | 8 | 0.9 | 0.3 | 5 | — | dispel, enemy |
| `consume_venom` | Consume Venom | venom | 9 | 0.9 | 0.4 | 5 | — | consume_venom 18, enemy |

- Two **standard** spells are carried by every player in every match, never drafted, never learned, never charged MP (`server:modules/spell_system/spell.go:18-22`, `server:modules/spellbook/rpc.go:156-162`, `server:modules/match/engine/state/player_state.go:421-431`).
- Six **starter** spells form the creation pool: a new character picks 3 (Human 4) distinct starters; only those become ownership rows. Unchosen starters cost 5 MP later like every other selectable spell (`server:modules/character/validate.go:106-129`, `server:modules/character/db.go:49-75`).
- `type` (`attack`/`defense`/`support`) and `icon` are presentation metadata and are not validated beyond being present in the YAML.
- `nature` and `incantation` are lore metadata, see [[spell-lore]].

## YAML schema

One spell per file, one YAML document per file, `.yaml` or `.yml`, walked recursively (`server:modules/spell_system/registry.go:96-146`). Unknown keys are rejected (`KnownFields(true)`). Struct: `server:modules/spell_system/spell.go:5-32`, `server:modules/spell_system/effect.go:3-10`, `server:modules/spell_system/identity.go:14-17`.

| Field | Type | Notes |
|---|---|---|
| `id`, `name`, `school` | string | required, non-empty |
| `type` | string | free text (`attack`, `defense`, `support` in use) |
| `nature` | string | one of `arcana`, `ember`, `vitality`, `venom`, `hex` |
| `incantation` | list of 2–3 words | action + nature word + optional modifier |
| `icon` | string | asset key |
| `description` | string | shown in spellbook |
| `flavor` | string | optional |
| `assets` | object | optional icon/vfx/sound descriptions |
| `casting_time`, `recovery_time`, `travel_time` | float seconds | 0–180, multiples of 0.1 |
| `mana_cost`, `magic_point_cost` | int | ≥ 0; explicit, zero is a real cost |
| `level_requirement` | int | ≥ 1 |
| `standard` | bool | omitted = false |
| `starter` | bool | **must be written explicitly** in every file |
| `effects[]` | list | `type`, `value`, `duration`, `delay`, `interval`, `target` |

### Validation rules (`server:modules/spell_system/registry.go:154-248,250-259,261-293,299-323`)

- Catalog non-empty; ids unique; incantation phrase unique across the catalog; nature/incantation grammar valid (see [[spell-lore]]).
- Every time field (cast, recovery, travel, duration, delay, interval) is finite, 0 ≤ t ≤ 180 and aligned to a 100 ms tick.
- At least one effect; `target` is `self` or `enemy`; all effects of one spell share the same target.
- Effect `value` is a non-negative integer ≤ 2^31−1.
- Effect kinds must be one of the 11 in the table below.
- Effect shape: `heal`, `cure`, `regeneration`, `shield`, `reflection` must target `self`; all others must target `enemy`. `poison`/`regeneration`: duration > 0, interval > 0, interval < duration, delay = 0. `paralyze`/`shield`/`reflection`: duration > 0, interval = delay = 0. `delayed_hex`: delay > 0, duration = interval = 0. Instant kinds: duration = interval = 0.
- Exactly 2 standard spells and they must be `magic_arrow` and `mirror_reflection`; a standard spell cannot be a starter and must have `magic_point_cost: 0`.
- Exactly 6 starter spells.

## Loader and registry

- Startup loads `/nakama/spells` (`server:modules/spell_system/init.go:36`). Docker Compose mounts `./data/spells` there (`server:docker-compose.yml:56`); the image copies it (`server:Dockerfile:21`). Replace the whole directory when deploying.
- `LoadFromDirectory` parses all files, validates the set, and only then swaps the registry (`server:modules/spell_system/registry.go:136-141`).
- Global registry `spell_system.Spells`. API: `Get`, `Exists`, `GetAll` (sorted by id), `GetStarter`, `GetStandard` (sorted by id, stable bar order), `IsStandard`, `GetByLevelRequirement` (`server:modules/spell_system/registry.go:20-59`, `server:modules/spell_system/standard.go`).
- Millisecond helpers for client payloads: `CastTimeMillis`, `RecoveryTimeMillis`, `TravelTimeMillis` (`server:modules/spell_system/spell.go:42-52`). Character details report these in ms; the catalog RPCs return seconds.

RPCs registered by the module (`server:modules/spell_system/init.go`): `get_spell_lore`, `get_entry_spells` (returns the six starters), `get_my_spells` (owned + both standards, optional `school`/`limit`), `get_spell`, `get_spell_details_yaml`. Payload shapes are in [[rpcs]].

## Effect kinds and handlers

Registered in `server:modules/spell_system/spell_effects/effect_handlers/registry.go:5-17`; all 11 kinds have a handler. Interface: `OnStart`, `OnTick`, `OnEnd` (`server:modules/spell_system/spell_effects/engine.go:9-13`).

| Kind | Handler file | Behaviour |
|---|---|---|
| `damage` | `damage.go` | instant: `value` damage at impact (no scaling, `calculateDamage` returns the raw value) |
| `heal` | `heal.go` | instant: heal caster by `value`, capped at max HP; periodic path skipped while poisoned |
| `poison` | `poison.go` | status; stops meditation on apply; each pulse deals `value` |
| `regeneration` | `tactical.go` | status; each pulse heals `value` unless target is poisoned (blocked pulses are lost) |
| `shield` | `shield.go` | status; sets `Shield = value`; cleared on end; depleted when damage exceeds it |
| `reflection` | `reflection.go` | status marker; consumed in `applySpell` (see below) |
| `paralyze` | `paralyze.go` | status; refused if already paralysed or immune; interrupts an in-flight cast (no mana refund, recovery restarts from interruption) |
| `cure` | `cure.go` | removes one: `delayed_hex` first, else `poison`; nothing to remove = no effect |
| `delayed_hex` | `tactical.go` | status marker with `due_tick`; detonates for `value` at delay, then removed |
| `dispel` | `tactical.go` | removes one from target: `reflection`, else `shield`, else `regeneration` |
| `consume_venom` | `tactical.go` | if target carries poison owned by the (final) caster: remove it and deal `value`; otherwise nothing |

Reflection is resolved before effects are scheduled (`server:modules/match/engine/phase/game/apply_spell_effect.go:20-33`): a hostile package hitting a target with `reflection` consumes that instance, swaps caster and target, and emits `spell_reflected`. The swapped package is applied directly, so it can never consume a second mirror. Self-target spells never reflect. Hex detonation is a queued tick on the target, so only the hex application reflects.

## Effect queue semantics

`server:modules/spell_system/spell_effects/queue.go`, `engine.go`, `events.go`.

- **One instance per status per target.** `Add` drops a status (poison, paralyze, shield, reflection, regeneration, delayed_hex) if the target already carries that kind. No refresh, no stacking, no mana refund (`queue.go:53-71`).
- **Paralyze immunity.** A paralyze scheduled while `ParalyzeImmuneUntil` is in the future is dropped (`queue.go:72-74`); on removal of a paralyze the target becomes immune for 3 s (`queue.go:124-126`).
- **Instances.** Each queued effect gets a monotonic `ID`; statuses copy it to the player's public `Effect` as `effect_instance_id` with `owner_id`, `start_tick`, `end_tick`, `remaining` and `due_tick` for hex (`queue.go:91-106`).
- **Scheduling.** `start = impact + delay`, `end = start + duration`, first pulse at `start + interval`. Hex: starts immediately, `end = due = impact + delay` (`engine.go:76-91`).
- **Expire** runs before casts release; removes started statuses whose `end` has passed, except hex (`queue.go:138-158`).
- **Process** repeatedly picks the earliest due item among scheduled impacts and effect pulses. Ties: earlier time, then seat order (player ids sorted ascending; on odd ticks the order is reversed), then sequence id (`queue.go:159-230`). A periodic pulse landing exactly on `end` is removed instead of pulsing, so poison and regeneration produce 5 pulses at seconds 1–5.
- **Removal** (`queue.go:111-135`) runs the handler's `OnEnd`, deletes the player effect, emits `effect_removed` with a reason: `expired`, `cleansed`, `dispelled`, `consumed`, `depleted` (shield broken by damage, from `player_state.go:125-146`), `match_ended`. Removing an instance cancels all its future pulses and detonation.
- **Damage/heal** go through `EffectContext.DealDamage`/`Heal`, which emit `damage` (with `absorbed`) and `heal` (with `overheal`) lifecycle events (`events.go:33-63`).
- `cast_interrupted` and `meditation_stopped` are emitted by the engine wrapper when a handler changed those states (`engine.go:27-46`).

The per-tick order in which the game phase drives the queue is documented in [[progression]] (section "Combat rules"); events and payloads in [[combat-v2]].

## Adding a spell

1. Copy an existing YAML in `data/spells/`, choose a unique stable id, set `starter` explicitly, set `nature` and a unique `incantation` (see [[spell-lore]]).
2. Keep the invariants: exactly two standards (`magic_arrow`, `mirror_reflection`) and exactly six starters.
3. A new combination of existing effect kinds needs data and scenario tests. A new effect kind additionally needs a handler, a line in `RegisterAllHandlers`, an entry in `legalEffects`, a shape rule in `validateEffectShape`, and `isStatus` if it is a status.
4. Bump `CatalogVersion` when balance values change and re-run the balance baseline (below).
5. Run:

```sh
go test ./modules/spell_system/... ./modules/match/engine/phase/game ./cmd/duel-sim
go test ./...
go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim
```

CI runs the last two before building the image (`server:.github/workflows/docker-publish.yml:26-27`). Relevant test files: `modules/spell_system/registry_test.go`, `identity_test.go`, `standard_test.go`, `version_test.go`, `db_test.go`; `modules/spell_system/spell_effects/effect_handlers/duel_effects_test.go`, `paralyze_test.go`; `modules/match/engine/phase/game/duel_test.go`, `scenarios_test.go`, `catalog_scenarios_test.go`; `modules/character/starter_spells_test.go`; `cmd/duel-sim/main_test.go`.

## Balance measurement

`cmd/duel-sim` runs the production `GamePhaseState.Advance` without Nakama or SQL (`server:cmd/duel-sim/main.go`). Flags: `-catalog` (default `data/spells`), `-seed` (42), `-matches` (100; 1–100000), `-out`. `make test-balance` runs 1000 matches with seed 42 (`server:Makefile:263-264`).

Simulation setup: both seats are bots at 200 HP / 100 mana; races rotate through the six ids; a Human seat gets 7 random non-standard spells, others 6, plus both standards; policies rotate `pressure`, `sustain`, `control` (`server:modules/match/engine/phase/game/ai.go`), which react only to casts/effects visible for at least 400 ms. Timeouts are counted at 1800 ticks (180 s). Invariants HP ∈ [0,200], mana ∈ [0,100] are asserted every tick.

Recorded run (catalog `duel_v2.1`, 2026-09-06, seed 42, 1000 duels, from `balance-v2.json`):

| Measurement | Result |
|---|---:|
| Median duration | 130.3 s |
| p90 duration | 180 s |
| Wins by seat | 294 / 301 |
| Draws | 405 |
| Timeouts | 404 |
| Interrupted casts | 2082 |

All 14 spells were cast. The `duel_v2.2` amendment changed only creation rules, not any catalog value, so the numbers still describe the current values; the run was not repeated for `duel_v2.2` (unverified). The 40% timeout rate is an open balance concern; bot results do not establish competitive balance. Remaining balance work is tracked in [[2026-09-05-spell-system-redesign]].

## Source of truth in code

- `server:data/spells/*.yaml` — the 14 definitions
- `server:modules/spell_system/spell.go`, `effect.go`, `identity.go`, `types.go` — schema and effect kinds
- `server:modules/spell_system/registry.go` — loader and validation rules
- `server:modules/spell_system/standard.go`, `version.go` — standard set, version constants
- `server:modules/spell_system/spell_effects/queue.go`, `engine.go`, `events.go` — effect queue, scheduling, lifecycle events
- `server:modules/spell_system/spell_effects/effect_handlers/*.go` — the 11 handlers
- `server:modules/match/engine/phase/game/apply_spell_effect.go` — reflection resolution
- `server:cmd/duel-sim/main.go`, `server:modules/match/engine/phase/game/ai.go` — balance simulator and bot policies
- `client:Core/Spells/Spell.cs`, `client:Core/Spells/StandardSpells.cs` — client-side spell model and standard-spell handling
