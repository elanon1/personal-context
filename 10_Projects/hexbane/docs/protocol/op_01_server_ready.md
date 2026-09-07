---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, server-ready]
sources: ["client:docs/opcodes/op_01_server_ready.md", "server:docs/opcodes/op_01_server_ready.md"]
---

# Opcode 1 — ServerReady

| | |
|---|---|
| Server const / client enum | `OP_SERVER_READY` / `SERVER_READY` |
| Direction | Server → client, broadcast, unreliable |
| Phase | `connecting` → `lobby_picking` transition |
| Sender | `server:modules/match/engine/phase/connecting/phase.go:107-113` |
| Client handler | `client:Application/Match/Incoming/ServerReady/ServerReadyHandler.cs` |

## When

Once, on the tick where every entry in `ConnectingPhaseState.Players` is ready (bots are ready from the start, `phase.go:58`). Sent immediately before `TransitionTo(PhaseLobbyPicking)` with a random first-pick player (`phase.go:112`).

## Payload

Empty (`make([]byte, 0)`).

## Client behaviour

If `MatchState.Status == Connecting` → `ChangeStatus(Lobby)`; the lobby scene then sends `CLIENT_READY` with `event_name:"lobby_ready"` (`client:Game/ScenesV3/Lobby/LobbyScreen.cs:169`) to receive [[op_70_lobby_spellbook_spells]].

## Source of truth in code
- `server:modules/match/engine/phase/connecting/phase.go` — readiness check and broadcast
- `client:Application/Match/Incoming/ServerReady/ServerReadyHandler.cs` — status transition
