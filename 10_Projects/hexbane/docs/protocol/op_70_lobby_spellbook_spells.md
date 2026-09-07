---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, lobby-spellbook-spells]
sources: ["client:docs/opcodes/op_70_lobby_spellbook_spells.md", "server:docs/opcodes/op_70_lobby_spellbook_spells.md"]
---

# Opcode 70 — LobbySpellbookSpells

| | |
|---|---|
| Server const / client enum | `OpLobbySpellbookSpells` / `LOBBY_SPELLBOK_SPELLS` (typo is in the enum) |
| Direction | Server → client, unicast per human player |
| Phase | `lobby_picking` |
| Sender | `server:modules/match/engine/phase/lobby/phase.go:159-185` (`GetAvailableSpells`), sent at `:259` and `:324` |
| Client handler | `client:Application/Match/Incoming/LobbySpellbookSpells/LobbySpellbookSpellsHandler.cs` |

## When

1. In reply to [[op_02_client_ready]] with `event_name:"lobby_ready"` (unicast, together with opcodes 3 and 5).
2. On any tick where `SpellbookRefresh` is set (after every pick, bot pick, auto-fill or turn hand-over), to each non-bot player whose `LobbyLoaded` flag is true. A client that never sent `lobby_ready` gets no refreshes.

## Payload

| Field | Type | Notes |
|---|---|---|
| `availableSpells` | [`{spell, used}`] | camelCase key is intentional (`LobbySpellbookSpellsMessage`, `phase.go:39-41`) |
| `spell` | [[shared-types#Spell]] | full catalog object |
| `used` | bool | already picked by this player in this draft |

Only `DraftableSpells()` are listed: the player's learned spellbook minus standard spells (`server:modules/match/engine/state/player_state.go:421-431`). `magic_arrow` and `mirror_reflection` never appear.

```json
{"availableSpells":[{"spell":{"id":"firebolt","name":"Firebolt","school":"fire","icon":"firebolt","casting_time":1.2,"recovery_time":0.6,"travel_time":0.4,"mana_cost":12,"effects":[{"type":"damage","value":24,"target":"enemy"}],"...":"..."},"used":true},
                    {"spell":{"id":"mend","...":"..."},"used":false}]}
```

## Client behaviour

Raises `GameEvents.LobbyOnSpellsLoaded(List<SpellBookSpell>)`; the lobby draft grid is built from this list, not from a catalog RPC. `Tests/DuelV2/Live.cs:37-41` asserts that standards never appear here.

## Source of truth in code
- `server:modules/match/engine/phase/lobby/phase.go` — `SpellbookSpell`, `LobbySpellbookSpellsMessage`, `GetAvailableSpells`
- `client:Application/Match/Incoming/LobbySpellbookSpells/LobbySpellbookSpellsMessage.cs` — DTO
