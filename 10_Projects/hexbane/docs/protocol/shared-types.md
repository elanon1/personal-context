---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, protocol, shared-types, payloads]
sources: ["client:docs/opcodes/shared_types.md", "server:docs/opcodes/shared_types.md", "server:docs/client/combat-v2.md"]
---

# Shared payload types

Field names are the Go `json` tags; the client column is the C# `JsonPropertyName` DTO. Only types that still appear on the wire are listed.

## Version

`server:modules/spell_system/version.go`. Embedded in [[op_00_match_entry_data]] (plus `tick_ms`), [[op_32_combat_snapshot]], and catalog/character RPC responses (`CurrentVersion()`, see [[rpcs]]).

| Field | Value |
|---|---|
| `combat_protocol` | `2` |
| `ruleset_id` | `"duel_v2"` |
| `catalog_version` | `"duel_v2.3"` |
| `tick_ms` (match entry only) | `100` |

Client check: `DuelVersion.Supported` (`client:Core/Match/DuelV2.cs:8-12`).

## Spell

`server:modules/spell_system/spell.go` (`type Spell struct`). Used inside `PrivatePlayerView.spells/standard_spells` ([[op_00_match_entry_data]], [[op_10_game_data]]) and `availableSpells[].spell` ([[op_70_lobby_spellbook_spells]]).

| Field | Type | Notes |
|---|---|---|
| `id` | string | one of the 14 catalog ids (`server:data/spells/*.yaml`) |
| `name`, `description` | string | |
| `flavor` | string | omitempty |
| `type`, `school`, `icon` | string | |
| `nature` | string | from embedded `Identity` |
| `incantation` | [string] | |
| `casting_time`, `recovery_time`, `travel_time` | float64 (**seconds**) | |
| `mana_cost`, `magic_point_cost`, `level_requirement` | int | |
| `standard` | bool | omitempty; true only for `magic_arrow`, `mirror_reflection` |
| `starter` | bool | creation-eligibility flag |
| `effects` | [Effect] | see below |
| `assets` | `{icon?, vfx?, sound?}` | omitempty; content not verified here |

Client DTO `client:Core/Spells/Spell.cs` maps `casting_time` → `CastingTimeSec` and also declares `cast_time` (ms, spellbook RPC shape), `icon_path`, `invocation`, `animation`, `visual_key`. None of the latter four are emitted by the match engine; `visual_key`/`animation` are client-only (see [[spell-visual-key]]). Old docs listed `level` and `effect` string fields: they do not exist on the struct.

Match spell views carry effective `mana_cost` and `casting_time` for the recipient, including opcode 70 draft entries. Recovery/travel/effect definitions retain catalog values. The server projects copies and never mutates shared catalog entries.

## Effect (spell effect definition)

`server:modules/spell_system/spell.go` (`type Effect struct`).

| Field | Type |
|---|---|
| `type` | EffectType |
| `value` | float64 |
| `duration`, `delay`, `interval` | float64 (seconds) |
| `target` | `"self"` or `"enemy"` (`server:modules/spell_system/types.go:31-32`) |

### EffectType values (`server:modules/spell_system/types.go:15-25`)

`damage`, `heal`, `poison`, `reflection`, `shield`, `paralyze`, `cure`, `delayed_hex`, `regeneration`, `dispel`, `consume_venom`.

Status effects that appear in a player's `effects` list (`isStatus`, `server:modules/spell_system/spell_effects/queue.go:46-52`): `poison`, `paralyze`, `shield`, `reflection`, `regeneration`, `delayed_hex`. The old `stun`, `slowdown`, `absorb`, `mana_drain` values are gone on the server; the client's `EffectType` constants (`client:Core/Spells/Effect.cs:35-47`) still list them (dead).

## PrivatePlayerView

`server:modules/match/engine/state/player_state/types.go:18-41`; built by `ToPrivateView` (`state/player_state.go:251`). Client: `client:Core/Match/PlayerTypes.cs:26`.

| Field | Type | Notes |
|---|---|---|
| `user_id`, `username`, `race_id` | string | `race_id` ∈ human, elf, dark_elf, shadow, gnome, orc (bot: random) |
| `health`, `max_health` | int | current HP / STR-and-race-derived maximum; see [[combat-stat-rules]] |
| `mana`, `max_mana` | int | current mana / INT-derived maximum |
| `spell_slots` | int | draftable this match = min(learned non-standard spells, entitlement) |
| `max_spell_slots` | int | character entitlement (`characters.spell_slots`) |
| `spells` | [Spell] | `SelectedSpells`: standards first, then drafted picks |
| `standard_spells` | [Spell] | always the two standards |
| `is_alive` | bool | |

## PublicPlayerView

`types.go:8-16`; `ToPublicView` (`player_state.go:241`). Client: `PlayerTypes.cs:7`.

| Field | Type | Notes |
|---|---|---|
| `user_id`, `username`, `race_id` | string | |
| `spell_slots` | int | |
| `spells` | [string] | tag exists, never populated → `null` on the wire |
| `is_alive` | bool | |

## PlayerSnapshot

`server:modules/match/engine/state/player_state/snapshot.go`; client `client:Core/Match/PlayerSnapshot.cs` (1:1). Appears as `players[id].state` in [[op_32_combat_snapshot]].

| Field | Type |
|---|---|
| `user_id` | string |
| `mana`, `mana_max`, `hp`, `hp_max`, `shield` | int |
| `casting`, `meditating`, `paralyzed`, `poisoned` | bool |
| `effects` | [ActiveEffect] |

## ActiveEffect

`server:modules/match/engine/state/player_state/types.go:50-62` (`Effect`); client `client:Core/Spells/Effect.cs` (1:1).

| Field | Type | Notes |
|---|---|---|
| `effect_instance_id` | uint64 | queue instance id; client identity key (`Effect.Identity`) |
| `owner_id` | string | caster |
| `key` | EffectType | engine name (`shield`, `reflection`, `paralyze`, `poison`, `regeneration`, `delayed_hex`) |
| `effect`, `icon` | string | display name / icon id (same as key today) |
| `start_tick`, `end_tick` | int64 | match ticks |
| `due_tick` | int64 | omitempty; hex detonation tick (= `end_tick`) |
| `remaining` | int | shield: current shield HP (patched in `GetSnapshot`); others: effect value |
| `started_at`, `remove_after` | RFC3339 | logical clock (1970-based); do not compare with wall-clock |

## CombatCommand / CombatEvent / snapshot structs

See [[op_29_combat_command]], [[op_31_combat_event]], [[op_32_combat_snapshot]] and [[combat-v2]].

## Removed types

`SpellStatus`, `SpellReason`, `SpellCastingStatus`, `MatchLog` on the wire, `EffectRemovalEvent` as a message: all belonged to retired opcodes 11–15/21–28 and no server type publishes them any more. `MatchLog`/`EffectRemovalEvent` still exist internally (`server:modules/match/engine/state/match_log.go`) but are drained and discarded each tick (`phase/game/phase.go:202-204`).

## Source of truth in code
- `server:modules/spell_system/spell.go`, `types.go`, `version.go` — Spell, Effect, EffectType, Version
- `server:modules/match/engine/state/player_state/{types,snapshot}.go` — views, snapshot, active effect
- `server:modules/match/engine/state/player_state.go` — `ToPrivateView`, `ToPublicView`, `GetSnapshot`
- `client:Core/Match/{PlayerTypes,PlayerSnapshot,DuelV2}.cs`, `client:Core/Spells/{Spell,Effect}.cs` — client DTOs
