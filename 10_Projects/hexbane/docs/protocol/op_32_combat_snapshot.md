---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, combat-snapshot]
sources: ["server:docs/client/combat-v2.md", "client:docs/opcodes/duel-v2.md"]
---

# Opcode 32 — CombatSnapshot

| | |
|---|---|
| Server const / client enum | `OpCombatSnapshot` / `COMBAT_SNAPSHOT` |
| Direction | Server → client, unicast per presence, reliable |
| Phase | `combat` (every 2 ticks = 200 ms, and on the final tick); any rejoin while in `combat` |
| Senders | `server:modules/match/engine/phase/game/phase.go:321-329`; `server:modules/match/engine/core/rejoin.go:16-25` |
| Client handler | `client:Application/Match/DuelProtocol.cs:41-56` → `DuelState.Apply` |

## Payload (`Snapshot`, `phase.go:336-351`)

| Field | Type | Notes |
|---|---|---|
| `combat_protocol`, `ruleset_id`, `catalog_version` | int, string, string | `2`, `duel_v2`, `duel_v2.2` |
| `server_tick` | int64 | ticks elapsed in combat |
| `event_seq` | uint64 | last emitted event sequence (watermark) |
| `last_client_seq` | uint64 | recipient's last accepted `client_seq`; resume above it after reconnect |
| `time_remaining` | int64 (s) | `180 - server_tick/10` |
| `over` | bool | |
| `players` | map user_id → player | both players |
| `queued` | `CombatCommand` or `null` | **recipient's** waiting action only |
| `pending_impacts` | [`PendingImpact`] | public in-flight spells: `owner_id`, `target_id`, `spell_id`, `action_id`, `due_tick` |

Per player:

| Field | Type | Notes |
|---|---|---|
| `state` | [[shared-types#PlayerSnapshot]] | HP/mana/shield/flags/effects; shield effect's `remaining` is the live shield value |
| `action` | `{action_id, spell_id, cast_end_tick}` or `null` | current cast |
| `recovery_end_tick` | int64 | 0 = none |
| `meditation_ready_tick` | int64 | warm-up end (0.8 s after start) |
| `immune_until_tick` | int64 | paralysis immunity (3 s after paralysis ends, `spell_effects/queue.go:125`) |

```json
{"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2","server_tick":100,"event_seq":30,"last_client_seq":17,"time_remaining":170,"over":false,
 "players":{"u1":{"state":{"user_id":"u1","hp":190,"hp_max":200,"mana":80,"mana_max":100,"shield":12,"casting":true,"meditating":false,"paralyzed":false,"poisoned":false,
                            "effects":[{"key":"shield","effect":"shield","icon":"shield","effect_instance_id":9,"owner_id":"u1","start_tick":90,"end_tick":150,"remaining":12,"started_at":"1970-01-01T00:00:09Z","remove_after":"1970-01-01T00:00:15Z"}]},
                   "action":{"action_id":"3","spell_id":"firebolt","cast_end_tick":110},"recovery_end_tick":114,"meditation_ready_tick":0,"immune_until_tick":0},
            "u2":{"state":{"...":"..."},"action":null,"recovery_end_tick":0,"meditation_ready_tick":0,"immune_until_tick":0}},
 "queued":{"client_seq":17,"kind":"meditate"},
 "pending_impacts":[{"owner_id":"u1","target_id":"u2","spell_id":"firebolt","action_id":"3","due_tick":113}]}
```

`started_at` / `remove_after` inside effects are the logical clock (`time.Unix(0,0) + ticks×100 ms`, `phase.go:84-86`) serialized as RFC3339, hence 1970 dates. Never compare them to the device clock; use `start_tick`/`end_tick`.

## Client behaviour (`DuelState.Apply`, `client:Core/Match/DuelV2.cs:96-101`)

1. Unsupported version → `InvalidOperationException` → `DuelProtocol.Fail`.
2. Stale (`server_tick` lower, or equal with lower `event_seq`) → ignored, not published.
3. Otherwise replaces the whole snapshot, raises the event watermark to `event_seq`, raises the local command sequence to `last_client_seq`, restarts the interpolation stopwatch.

Then `DuelProtocol.Receive` raises `OnPlayerUpdate(state)` and `OnPlayerEffectUpdate(id, effects)` per player, `OnGameTimeChange(time_remaining)`, `OnDuelSnapshot`. Countdowns are interpolated from `server_tick` (`DuelState.Tick`), capped at 500 ms of extrapolation.

## Source of truth in code
- `server:modules/match/engine/phase/game/phase.go:336-351` — payload
- `server:modules/match/engine/core/rejoin.go` — rejoin snapshot
- `client:Core/Match/DuelV2.cs` — `DuelSnapshot`, `DuelPlayer`, `DuelImpact`, `DuelState`
