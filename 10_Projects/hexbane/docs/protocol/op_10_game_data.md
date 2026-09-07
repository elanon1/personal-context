---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, game-data]
sources: ["client:docs/opcodes/op_10_game_data.md", "server:docs/opcodes/op_10_game_data.md", "server:docs/client/combat-v2.md"]
---

# Opcode 10 — GameData

| | |
|---|---|
| Server const / client enum | `OpGameData` / `GAME_ENTRY_DATA` |
| Direction | Server → client, unicast to the requesting player |
| Phase | `game_countdown` |
| Sender | `server:modules/match/engine/phase/game_countdown/phase.go:44-60,99` |
| Client handler | `client:Application/Match/Incoming/GameEntryData/GameEntryDataHandler.cs` |

## When

In reply to any [[op_02_client_ready]] received during `game_countdown` (the client sends `game_countdown_ready` on [[op_07_game_ready]], and the HUD sends `game_hud_ready`; both may be answered). Presence is looked up by the payload's `user_id`.

## Payload

| Field | Type | Notes |
|---|---|---|
| `me` | [[shared-types#PrivatePlayerView]] | full loadout: `spells` = standard spells + drafted spells (in that order), `standard_spells` repeated |
| `enemy` | [[shared-types#PrivatePlayerView]] | **also the private view** (`phase.go:51`, marked `todo change to public?`), so the enemy's full spell list and HP/mana are disclosed |

```json
{"me":{"user_id":"u1","username":"Ayla","race_id":"elf","health":200,"max_health":200,"mana":100,"max_mana":100,
       "spell_slots":3,"max_spell_slots":6,
       "spells":[{"id":"magic_arrow","standard":true,"casting_time":0.6,"recovery_time":0.4,"travel_time":0.3,"mana_cost":5,"effects":[{"type":"damage","value":12,"target":"enemy"}],"...":"..."},{"id":"mirror_reflection","standard":true,"...":"..."},{"id":"firebolt","...":"..."}],
       "standard_spells":[{"id":"magic_arrow","...":"..."},{"id":"mirror_reflection","...":"..."}],"is_alive":true},
 "enemy":{"user_id":"0000","username":"Bot","race_id":"orc","spells":[ "..." ],"...":"..."}}
```

Spell objects are the catalog `spell_system.Spell` (see [[shared-types#Spell]]); times are **seconds**.

## Client behaviour

Stores both views in `MatchContext.PendingGameLoadMe/Enemy` (so a HUD that spawns later can pick them up, `client:Game/Autoloads/MatchContext.cs:35`) and raises `GameEvents.OnGameLoad(me, enemy)`. The HUD deduplicates `spells` + `standard_spells` by id (`DuelLoadout.DistinctById`, `client:Core/Match/DuelV2.cs:109`) and draws standards in their own frames. `DuelProtocol.PresentationSpell` (`client:Application/Match/DuelProtocol.cs:65`) later resolves `spell_id`s from events/snapshots against these loadouts, per owner.

## Source of truth in code
- `server:modules/match/engine/phase/game_countdown/phase.go` — `GetGameData`
- `server:modules/match/engine/state/player_state.go:251-266` — `ToPrivateView`
- `client:Application/Match/Incoming/GameEntryData/GameEntryDataMessage.cs` — DTO
