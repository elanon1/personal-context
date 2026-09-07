---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, lobby-update]
sources: ["client:docs/opcodes/op_03_lobby_update.md", "server:docs/opcodes/op_03_lobby_update.md"]
---

# Opcode 3 — LobbyUpdate

| | |
|---|---|
| Server const / client enum | `OpLobbyUpdate` / `LOBBY_UPDATE` |
| Direction | Server → client, broadcast (unicast once on `lobby_ready`) |
| Phase | `lobby_picking` (draft state), `lobby_countdown` (timer only) |
| Senders | `server:modules/match/engine/phase/lobby/phase.go:194,218,248,304,326`; `server:.../phase/lobby_countdown/phase.go:47` |
| Client handler | `client:Application/Match/Incoming/LobbyUpdate/LobbyUpdateHandler.cs` |

## When

`lobby_picking`:
- once per second (timer decrement of the drafting player only, `phase.go:190-195`);
- immediately after a human pick (`:304`), a bot pick (`:248`), or a timer expiry / auto-fill (`:218`);
- unicast to a player who just sent `lobby_ready` (`:326`).

`lobby_countdown`: once per second (`lobby_countdown/phase.go:42-48`), **not** every tick.

## Payload (`lobby_picking`)

| Field | Type | Notes |
|---|---|---|
| `time_remaining` | int64 (s) | seconds left for the player whose turn it is |
| `player_timers` | map user_id → int64 | per-player remaining draft seconds (each starts at 35) |
| `draft_turn_user_id` | string | who picks now |
| `spells_selected` | map user_id → [spell_id] | picks so far |
| `spell_slots` | map user_id → int | draftable slots per player (min(learned, entitlement), `server:modules/match/engine/state/state.go:129-144`) |
| `slots_left` | map user_id → int | `spell_slots - len(spells_selected)` |

```json
{"time_remaining":31,"player_timers":{"u1":31,"u2":35},"draft_turn_user_id":"u1",
 "spells_selected":{"u1":["firebolt"],"u2":[]},"spell_slots":{"u1":3,"u2":3},"slots_left":{"u1":2,"u2":3}}
```

## Payload (`lobby_countdown`)

```json
{"time_remaining":14}
```

Only `time_remaining` (15 → 0).

## Client behaviour

`LobbyUpdateHandler.cs:18`: if `draft_turn_user_id` is empty (countdown variant) only `LobbyTimeUpdate` is emitted. Otherwise, while status is `Lobby`, emits `LobbyTimeUpdate` and `LobbyDraftUpdate(draftUserId, picks of that user)`. When every `slots_left` is 0 → `ChangeStatus(LobbySelected)`. The DTO ignores `player_timers` beyond parsing it.

## Source of truth in code
- `server:modules/match/engine/phase/lobby/phase.go:63-85` — `GetJsonPayload`
- `server:modules/match/engine/phase/lobby_countdown/phase.go:29-39` — countdown payload
- `client:Application/Match/Incoming/LobbyUpdate/LobbyUpdateMessage.cs` — DTO
