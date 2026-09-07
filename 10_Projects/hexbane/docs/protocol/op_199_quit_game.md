---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, quit-game]
sources: ["client:docs/opcodes/op_199_quit_game.md", "server:docs/opcodes/op_199_quit_game.md"]
---

# Opcode 199 — QuitGame

| | |
|---|---|
| Server const / client enum | `OpQuitGame` / `QuitGame` |
| Direction | Client → server |
| Phase | `combat_end` only (every other phase ignores it; `combat` logs "unsupported combat opcode") |
| Server handler | `server:modules/match/engine/phase/gameover/phase.go:362-369` |
| Client sender | `client:Application/Match/Outgoing/Quit/QuitCommandHandler.cs` |

## Payload

The client serializes `QuitCommand` as `{"MatchId":"…"}`; the server parses nothing and only sets `TerminateMatch = true`.

## Practical effect

None in the current flow: the game-over `Tick` already sets `TerminateMatch` on the same tick it sends [[op_50_game_over]] (`phase.go:176-177`), and the match loop returns `nil` on the next tick (`server:modules/match/engine/core/loop.go:45-48`). By the time the client could send 199 the match is gone; the client leaves via `LeaveMatchAsync` in its match managers anyway.

## Source of truth in code
- `server:modules/match/engine/phase/gameover/phase.go` — handler
- `client:Application/Match/Outgoing/Quit/QuitCommandHandler.cs` — sender
