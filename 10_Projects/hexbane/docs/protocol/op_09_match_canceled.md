---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, match-canceled]
sources: ["client:docs/opcodes/op_09_match_canceled.md", "server:docs/opcodes/op_09_match_canceled.md", "client:docs/opcodes/op_09_game_countdown.md"]
---

# Opcode 9 — MatchCanceled

| | |
|---|---|
| Server const / client enum | `OP_MATCH_CANCELED` / `MATCH_CANCELED` |
| Direction | Server → client, reliable |
| Phase | any join; pre-combat leave |
| Senders | `server:modules/match/normal_match/join.go:36`, `server:modules/match/ai_match/join.go:38`, `server:modules/match/normal_match/leave.go:43` |
| Client handler | `client:Application/Match/Incoming/MatchCanceled/MatchCanceledHandler.cs` |

Opcode 9 is **only** this message. The old "shared with GameCountdown" note is wrong: the countdown is opcode 16 ([[op_16_game_countdown]]).

## When / payload

| `reason` | Trigger | Recipients |
|---|---|---|
| `"player_setup_failed"` | `BuildPlayerState` failed on join (no character, spellbook load error) in `normal` or `ai_duel` | broadcast |
| `"opponent_left"` | a presence left a `normal` match during `connecting`, `lobby_picking`, `lobby_countdown`, `loading`, `game_countdown` (`leave.go:58-69`) | unicast to every remaining presence |

Both set `TerminateMatch = true`. A leave during `combat`/`combat_end` does not cancel; the match keeps running (bot keeps fighting; the absent player can rejoin, see [[combat-v2#Reconnect]]). `ai_duel` never sends `opponent_left` (its `MatchLeave` only removes the presence, `server:modules/match/ai_match/leave.go`); the match self-terminates after 5 s without presences.

```json
{"reason":"opponent_left"}
```

## Client behaviour

Logs, `ChangeStatus(Idle)`, `MatchContext.Reset()`, `OnMatchCanceled(reason ?? "opponent_left")`.

## Source of truth in code
- `server:modules/match/normal_match/leave.go` — pre-combat cancel
- `server:modules/match/normal_match/join.go`, `server:modules/match/ai_match/join.go` — setup failure
- `client:Application/Match/Incoming/MatchCanceled/MatchCanceledMessage.cs` — DTO
