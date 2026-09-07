---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, match-entry-data]
sources: ["client:docs/opcodes/op_00_match_entry_data.md", "server:docs/opcodes/op_00_match_entry_data.md", "server:docs/client/combat-v2.md"]
---

# Opcode 0 — MatchEntryData

| | |
|---|---|
| Server const / client enum | `OpMatchEntryData` / `MATCH_ENTRY_DATA` |
| Direction | Server → client, unicast per human player, unreliable |
| Phase | `connecting` |
| Sender | `server:modules/match/engine/phase/connecting/phase.go:76-104` |
| Client handler | `client:Application/Match/Incoming/MatchEntryData/MatchEntryDataHandler.cs` |

## When

Once per second (`ms.Tick % TicksPerSecond == 0`, not every tick) to each non-bot player with a presence, for as long as the match is in `connecting`. Repeats until both players are ready, so a client may receive it several times; the handler is idempotent (re-creates `Me`/`Enemy` and re-sends readiness).

## Payload

| Field | Type | Notes |
|---|---|---|
| `combat_protocol` | int | `2` (`server:modules/spell_system/version.go:4`) |
| `ruleset_id` | string | `"duel_v2"` |
| `catalog_version` | string | `"duel_v2.2"` |
| `tick_ms` | int | `100` |
| `me` | [[shared-types#PrivatePlayerView]] | own full view incl. `spells`, `standard_spells`, `spell_slots`, `max_spell_slots`, `race_id` |
| `enemy` | [[shared-types#PublicPlayerView]] | opponent public view (`spells` is always `null`: `ToPublicView` never fills it) |

```json
{"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2","tick_ms":100,
 "me":{"user_id":"u1","username":"Ayla","race_id":"elf","health":200,"max_health":200,"mana":100,"max_mana":100,
       "spell_slots":3,"max_spell_slots":6,"spells":[{"id":"magic_arrow","standard":true,"...":"..."}],"standard_spells":[{"id":"magic_arrow","...":"..."}],"is_alive":true},
 "enemy":{"user_id":"0000","username":"Bot","race_id":"orc","spell_slots":3,"spells":null,"is_alive":true}}
```

Only the standard spells are in `me.spells` at this point; drafted spells are appended during the lobby (`server:modules/match/engine/state/player_state.go:268`).

## Client behaviour

`MatchEntryDataHandler.cs:17` rejects the match (`DuelProtocol.Fail`, status `Error`, `OnMatchCanceled`) unless `DuelVersion.Supported(protocol, ruleset, catalog)` (`client:Core/Match/DuelV2.cs:10`) and `tick_ms == 100`. Otherwise it sets `Combat.ProtocolAccepted`, fills `MatchContext.Me/Enemy` (user id, name, slots, HP/mana) and dispatches `ClientReadyCommand(userId, "match_entry_data", matchId)` → [[op_02_client_ready]].

## Source of truth in code
- `server:modules/match/engine/phase/connecting/phase.go` — cadence and payload map
- `server:modules/match/engine/state/player_state/types.go` — `PrivatePlayerView` / `PublicPlayerView`
- `server:modules/spell_system/version.go` — version constants
- `client:Application/Match/Incoming/MatchEntryData/MatchEntryDataMessage.cs` — DTO
