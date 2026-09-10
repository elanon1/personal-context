---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, protocol, opcode, game-ready]
sources: ["client:docs/opcodes/op_07_game_ready.md", "server:docs/opcodes/op_07_game_ready.md"]
---

# Opcode 7 — GameReady

| | |
|---|---|
| Server const / client enum | `OpGameReady` / `GAME_READY` |
| Direction | Server → client, broadcast |
| Phase | `loading` → `game_countdown` |
| Sender | `server:modules/match/engine/phase/loading/phase.go:23-32` |
| Client handler | `client:Application/Match/Incoming/GameReady/GameReadyHandler.cs` |

## When

After every non-bot participant sends protocol-2 `game_hud_ready`. The server broadcasts opcode 7 reliably and enters `game_countdown`. Loading times out after 30 seconds with opcode 9 `loading_timeout`; it never starts combat for an unready player.

## Payload

Empty.

## Client behaviour

Dispatches `ClientReadyCommand(userId, "game_countdown_ready", matchId)` → [[op_02_client_ready]], which the `game_countdown` phase answers with [[op_10_game_data]]. The HUD sends `game_hud_ready` after the arena's first rendered frame and completed scene transition; loading uses it to gate the countdown. The second acknowledgement here requests the loadout in `game_countdown`.

## Source of truth in code
- `server:modules/match/engine/phase/loading/phase.go` — readiness barrier and 30-second timeout
- `client:Application/Match/Incoming/GameReady/GameReadyHandler.cs` — re-request of game data
