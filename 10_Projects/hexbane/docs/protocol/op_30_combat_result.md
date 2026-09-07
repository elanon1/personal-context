---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, combat-result]
sources: ["server:docs/client/combat-v2.md", "client:docs/opcodes/duel-v2.md"]
---

# Opcode 30 — CombatResult

| | |
|---|---|
| Server const / client enum | `OpCombatResult` / `COMBAT_RESULT` |
| Direction | Server → client, unicast to the acting player, reliable |
| Phase | `combat`; also `connecting` for `protocol_rejected` |
| Senders | `server:modules/match/engine/phase/game/phase.go:312-316`; `server:modules/match/engine/phase/connecting/phase.go:122` |
| Client handler | `client:Application/Match/DuelProtocol.cs:38-64` (`Receive`) |

## Payload

Same struct as [[op_31_combat_event]] (`CombatEvent`, `phase.go:42-64`; all fields except `event_seq`, `server_tick`, `kind`, `player_id` are `omitempty`). Only these kinds travel on 30:

| `kind` | Extra fields | Meaning |
|---|---|---|
| `action_queued` | `client_seq`, `spell_id` (cast only) | command accepted and now the single waiting action |
| `queue_cleared` | `client_seq` | `clear_queue` applied |
| `action_rejected` | `client_seq`, `reason` | rejected at submit or at execution; reasons in [[combat-v2#Rejection reasons]] |
| `protocol_rejected` | `required_protocol` (=2) | connecting-phase handshake failure; **not** a `CombatEvent`, sent as a literal `{"kind":"protocol_rejected","required_protocol":2}` with no `event_seq` |

```json
{"event_seq":12,"server_tick":41,"kind":"action_queued","player_id":"u1","client_seq":7,"spell_id":"firebolt"}
{"event_seq":13,"server_tick":41,"kind":"action_rejected","player_id":"u1","client_seq":8,"reason":"insufficient_mana"}
```

Results consume an `event_seq` from the shared counter, so the public stream on 31 has gaps; clients must accept gaps.

## Client behaviour

`DuelProtocol.Receive`: `protocol_rejected` → `Fail(evt.Reason ?? "Unsupported combat version")` (the server sends `required_protocol`, not `reason`, so the fallback text is what appears). Anything else passes the `AcceptEvent(seq)` watermark (`client:Core/Match/DuelV2.cs:102`) and is raised as `GameEvents.OnDuelEvent`. The HUD logs `action_rejected` reasons and marks the queued slot from the next snapshot, not from this message.

## Source of truth in code
- `server:modules/match/engine/phase/game/phase.go:305-320` — routing of private kinds to 30
- `server:modules/match/engine/phase/connecting/phase.go:118-125` — handshake rejection
- `client:Application/Match/DuelProtocol.cs` — receive path
