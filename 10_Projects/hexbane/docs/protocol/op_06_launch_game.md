---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, protocol, opcode, launch-game]
sources: ["client:docs/opcodes/op_06_launch_game.md", "server:docs/opcodes/op_06_launch_game.md"]
---

# Opcode 6 — LaunchGame

| | |
|---|---|
| Server const / client enum | `OpLaunchGame` / `LOBBY_START_GAME` |
| Direction | Server → client, broadcast |
| Phase | `lobby_countdown` → `loading` |
| Sender | `server:modules/match/engine/phase/lobby_countdown/phase.go:50-55` |
| Client handler | `client:Application/Match/Incoming/LobbyStartGame/LobbyStartGameHandler.cs` |

## When

Once, on the tick where the 15 s lobby countdown reaches 0. The match then enters `loading`, which waits for all human arenas to acknowledge `game_hud_ready` before broadcasting [[op_07_game_ready]].

## Payload

Empty.

## Client behaviour

`ChangeStatus(InMatch)`; the scene manager loads `Game/ScenesV3/ReferenceDuel/MainReference.tscn` (`client:Game/Autoloads/SceneManager.cs:30`, key `normal_game`).

## Source of truth in code
- `server:modules/match/engine/phase/lobby_countdown/phase.go` — countdown and broadcast
- `client:Application/Match/Incoming/LobbyStartGame/LobbyStartGameHandler.cs` — status change
