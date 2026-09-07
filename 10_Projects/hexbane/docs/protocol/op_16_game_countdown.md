---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, game-countdown]
sources: ["client:docs/opcodes/op_09_game_countdown.md", "server:docs/opcodes/op_09_game_countdown.md"]
---

# Opcode 16 — GameCountdown

| | |
|---|---|
| Server const / client enum | `OpGameCountdown` / `GAME_COUNTDOWN` |
| Direction | Server → client, broadcast |
| Phase | `game_countdown` |
| Sender | `server:modules/match/engine/phase/game_countdown/phase.go:62-78` |
| Client handler | `client:Application/Match/Incoming/GameCountdown/GameCountdownHandler.cs` |

Value is **16** (`server:modules/match/engine/match_types/op_codes.go:24`, `client:Core/Common/Enums/Opcodes.cs:24`). The old docs filed it as opcode 9; that is stale.

## When

Once per second while `TicksLeft >= 0`. `GameCountdownDurationTicks = 2`, so three messages go out (`2`, `1`, `0`); on the second after `0` the phase transitions to `combat` (`phase.go:71-74`). During this phase the server also answers [[op_02_client_ready]] with [[op_10_game_data]].

## Payload

```json
{"time_remaining":2}
```

| Field | Type |
|---|---|
| `time_remaining` | int64 (s) |

## Client behaviour

Raises `GameEvents.OnGameCountdownChange(string)`; the HUD shows the countdown and unlocks input when it finishes.

## Source of truth in code
- `server:modules/match/engine/phase/game_countdown/phase.go` — cadence and payload
- `client:Application/Match/Incoming/GameCountdown/GameCountdownMessage.cs` — DTO
