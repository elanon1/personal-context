---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, match-declined]
sources: ["client:docs/opcodes/op_08_match_declined.md", "server:docs/opcodes/op_08_match_declined.md"]
---

# Opcode 8 — MatchDeclined

| | |
|---|---|
| Server const / client enum | `OP_MATCH_DECLINED` / `MATCH_DECLINED` |
| Direction | Server → client, broadcast to all current presences, unreliable |
| Phase | any (`normal` matches only; `ai_duel` ignores signals) |
| Sender | `server:modules/match/normal_match/signal.go:22-27` |
| Client handler | `client:Application/Match/Incoming/MatchDeclined/MatchDeclinedHandler.cs` |

## When

The RPC `decline_match` (`server:modules/match/normal_match/init.go:31-53`, payload `{"match_id":"…"}`) calls `nk.MatchSignal(matchId, "decline")`. `MatchSignal` broadcasts opcode 8 and sets `TerminateMatch`. The client calls that RPC from `client:Application/ArcaneDuel/Normal/MatchManager.cs:159` when the player refuses the found match; the decliner has usually not joined yet, so only the opponent receives the message.

## Payload

```json
{}
```

## Client behaviour

`ChangeStatus(Idle)`, `MatchContext.Reset()`, `GameEvents.OnMatchDeclined`.

## Source of truth in code
- `server:modules/match/normal_match/signal.go` — signal handling
- `server:modules/match/normal_match/init.go` — `decline_match` RPC
- `client:Application/ArcaneDuel/Normal/MatchManager.cs` — `DeclineMatch`
