---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
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

On the very first tick of `loading` (the phase lasts one tick and waits for nothing). Immediately followed by the transition to `game_countdown`.

## Payload

Empty.

## Client behaviour

Dispatches `ClientReadyCommand(userId, "game_countdown_ready", matchId)` → [[op_02_client_ready]], which the `game_countdown` phase answers with [[op_10_game_data]]. The HUD's own `game_hud_ready` readiness typically fires while the server is still in `loading` and is ignored there, which is why this handler exists (comment in `GameReadyHandler.cs:10-18`).

## Source of truth in code
- `server:modules/match/engine/phase/loading/phase.go` — one-tick phase
- `client:Application/Match/Incoming/GameReady/GameReadyHandler.cs` — re-request of game data
