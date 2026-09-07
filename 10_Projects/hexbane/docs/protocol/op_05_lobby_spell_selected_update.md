---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, lobby-spell-selected-update]
sources: ["client:docs/opcodes/op_05_lobby_spell_selected_update.md", "server:docs/opcodes/op_05_lobby_spell_selected_update.md"]
---

# Opcode 5 — LobbySpellSelectedUpdate

| | |
|---|---|
| Server const / client enum | `OpLobbySpellSelectedUpdate` / `LOBBY_SPELLS_SELECTED_UPDATE` |
| Direction | Server → client, broadcast (unicast once on `lobby_ready`) |
| Phase | `lobby_picking` |
| Sender | `server:modules/match/engine/phase/lobby/phase.go:87-119` (`GetSpellSelectedUpdatePayload`), sent at `:219,249,305,327` |
| Client handler | `client:Application/Match/Incoming/LobbySpellsPicked/LobbySpellsPickedHandler.cs` |

## When

Always together with [[op_03_lobby_update]] after a pick, a bot pick, an auto-fill, or on `lobby_ready` (unicast). Not part of the per-second timer broadcast.

## Payload

| Field | Type |
|---|---|
| `spells_selected` | map user_id → [`SpellInfo`] |
| `spell_slots` | map user_id → int |

`SpellInfo` (`server:modules/match/engine/phase/lobby/types.go:8-13`): `spell_id`, `name`, `description`, `icon`.

```json
{"spells_selected":{"u1":[{"spell_id":"firebolt","name":"Firebolt","description":"…","icon":"firebolt"}],"u2":[]},
 "spell_slots":{"u1":3,"u2":3}}
```

## Client behaviour

Raises `GameEvents.OnSpellsSelected(dictionary)`; the DTO's nested `Spell` class (`LobbySpellsPickedMessage.cs:29`) maps `spell_id` → `Id` and also declares `icon_path`, which the server never sends.

## Source of truth in code
- `server:modules/match/engine/phase/lobby/phase.go` — payload builder
- `client:Application/Match/Incoming/LobbySpellsPicked/LobbySpellsPickedMessage.cs` — DTO
