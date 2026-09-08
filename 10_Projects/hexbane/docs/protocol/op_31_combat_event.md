---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, protocol, opcode, combat-event]
sources: ["server:docs/client/combat-v2.md", "client:docs/opcodes/duel-v2.md"]
---

# Opcode 31 — CombatEvent

| | |
|---|---|
| Server const / client enum | `OpCombatEvent` / `COMBAT_EVENT` |
| Direction | Server → client, broadcast to all presences, reliable |
| Phase | `combat` |
| Sender | `server:modules/match/engine/phase/game/phase.go:317` (every tick, after `Advance`) |
| Client handler | `client:Application/Match/DuelProtocol.cs:57-63` → `GameEvents.OnDuelEvent`, plus `GameLog` signal for `damage`/`heal` |

## Payload (`CombatEvent`, `phase.go:42-64`)

| Field | Type | Present |
|---|---|---|
| `event_seq` | uint64 | always; shared counter with [[op_30_combat_result]] (gaps are normal) |
| `server_tick` | int64 | always; match tick (100 ms) at emission |
| `kind` | string | always |
| `player_id` | string | always (may be empty for `match_ended`) |
| `target_id` | string | impacts, effects, damage/heal |
| `client_seq` | uint64 | kinds tied to a command |
| `action_id` | string | cast identity (decimal counter as string) |
| `spell_id` | string | casts, impacts, effects |
| `reason` | string | `action_rejected`, `meditation_stopped`, `cast_interrupted`, `effect_removed`, `match_ended`; also `spell_impact:dodged`, `heal:passive_regeneration` |
| `cast_end_tick`, `recovery_end_tick` | int64 | `cast_started` |
| `due_tick` | int64 | `effect_applied` for `delayed_hex` |
| `start_tick`, `end_tick` | int64 | effect events |
| `effect_instance_id` | uint64 | effect events |
| `effect_kind` | string | effect events (engine names: `shield`, `reflection`, `paralyze`, `poison`, `regeneration`, `delayed_hex`) |
| `amount` | int | `damage` (HP actually lost), `heal` (HP restored), `mana_spent`, `mana_regenerated` |
| `absorbed` | int | `damage`: shield consumed |
| `overheal` | int | `heal`: wasted healing |
| `remaining` | int | `effect_applied`: effect value (shield capacity / hex or poison amount) |

Deadline fields are rounded up to the first authoritative 100 ms tick. Tick fields are match-relative ticks (`deadlineTick`, `phase.go:374`); `0` means no deadline.

## Kinds actually emitted

| `kind` | Emitted by | Notes |
|---|---|---|
| `mana_spent` | `phase.go:300` | at cast start, effective racial mana cost |
| `cast_started` | `:301` | with `client_seq`, `action_id`, `spell_id`, `cast_end_tick`, `recovery_end_tick` |
| `cast_released` | `:169` | cast time elapsed; impact scheduled after `travel_time` |
| `cast_interrupted` | `spell_effects/engine.go:32` | `reason:"paralyzed"`; the only interrupt source |
| `spell_impact` | `apply_spell_effect.go:38` | `player_id` = final caster, `target_id` = final target; `reason:dodged` means the whole hostile package missed after reflection |
| `spell_reflected` | `apply_spell_effect.go:32` | `player_id` = reflector (new owner), `target_id` = original caster; mirror removed with `consumed` |
| `effect_applied` | `spell_effects/queue.go:102` | status effects only (`isStatus`, `queue.go:46`) |
| `effect_removed` | `queue.go:128`, `events.go:44` | `reason` ∈ `expired`, `depleted`, `consumed`, `cleansed`, `dispelled`, `match_ended` |
| `damage` | `events.go:38` | `amount` + `absorbed` |
| `heal` | `events.go:58` | `amount` + `overheal` |
| `meditation_started` | `phase.go:282` | with `client_seq` |
| `meditation_stopped` | `:253` (no reason: mana full / regen tick), `:298` (`reason:"cast_started"`), `engine.go:40` (`reason` = effect type, e.g. `poison`, `paralyze`) | |
| `mana_regenerated` | `:251` | profile-derived passive mana plus active meditation after 0.8 s warm-up; see [[combat-stat-rules]] |
| `match_ended` | `:241` | `reason` ∈ `defeated`, `timeout`, `draw` |

Private kinds (`action_queued`, `queue_cleared`, `action_rejected`) go to [[op_30_combat_result]] instead (`phase.go:312`).

```json
{"event_seq":40,"server_tick":128,"kind":"cast_started","player_id":"u1","client_seq":7,"action_id":"3","spell_id":"firebolt","cast_end_tick":140,"recovery_end_tick":146}
{"event_seq":47,"server_tick":143,"kind":"damage","player_id":"u1","target_id":"u2","action_id":"3","spell_id":"firebolt","effect_kind":"damage","amount":18,"absorbed":6}
```

## Client behaviour

Events drive animation/log only; HP/mana are never mutated from events. `DuelProtocol.Receive` drops events at or below the watermark, raises `OnDuelEvent`, and emits the Godot `GameLog(kind, amount, target)` signal for `damage`/`heal`.

## Source of truth in code
- `server:modules/match/engine/phase/game/phase.go` — struct, `emit`, `Advance`, `Tick`
- `server:modules/spell_system/spell_effects/{events,queue,engine}.go` — effect-driven kinds
- `client:Core/Match/DuelV2.cs:55-77` — `DuelEvent` DTO (field names match 1:1)
