---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, lobby-spell-selected]
sources: ["client:docs/opcodes/op_04_lobby_spell_selected.md", "server:docs/opcodes/op_04_lobby_spell_selected.md"]
---

# Opcode 4 — LobbySpellSelected

| | |
|---|---|
| Server const / client enum | `OpLobbySpellSelected` / `LOBBY_SPELL_SELECTED` |
| Direction | Client → server |
| Phase | `lobby_picking` only (ignored elsewhere) |
| Server handler | `server:modules/match/engine/phase/lobby/phase.go:287-310` |
| Client sender | `client:Application/Match/Outgoing/SpellSelection/LobbySpellSelectionHandler.cs` |

## Payload

| Field | Type | Notes |
|---|---|---|
| `spell_id` | string | catalog id |
| `match_id` | string | sent by client, unused by server |

```json
{"spell_id":"firebolt","match_id":"…"}
```

Acting user is the message sender (`SelectSpell(data.GetUserId(), …)`).

## Server validation (`SelectSpell`, `phase.go:121-157`), each failure only logged, no reply

1. it is the sender's turn (`DraftTurnUserId`);
2. not already picked by this player;
3. not a standard spell (`spell_system.Spells.IsStandard`) — `magic_arrow` / `mirror_reflection` are never draftable;
4. player still has a free slot.

On success the spell is appended to the player's `SelectedSpells`, the turn moves (`determineNextTurn`, `lobby/utils.go:34`: alternate while both have slots, otherwise whoever still has slots), and `SpellbookRefresh` is set so [[op_70_lobby_spellbook_spells]] is re-sent next tick. Success or failure, the server then broadcasts [[op_03_lobby_update]] and [[op_05_lobby_spell_selected_update]]; when all slots are full it transitions to `lobby_countdown`.

Note the spell is **not** checked against the player's own spellbook here; `AddSpellToSpellbook` accepts any catalog id (`server:modules/match/engine/state/player_state.go:268`). See report.

## Source of truth in code
- `server:modules/match/engine/phase/lobby/types.go:3-6` — `SpellSelected` struct
- `server:modules/match/engine/phase/lobby/phase.go` — `SelectSpell`, `HandleMessage`
- `client:Application/Match/Outgoing/SpellSelection/LobbySpellSelectionCommand.cs` — payload
