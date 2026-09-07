---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, combat-command]
sources: ["server:docs/client/combat-v2.md", "client:docs/opcodes/duel-v2.md"]
---

# Opcode 29 — CombatCommand

| | |
|---|---|
| Server const / client enum | `OpCombatCommand` / `COMBAT_COMMAND` |
| Direction | Client → server |
| Phase | `combat` only (any other opcode in combat is logged as "unsupported combat opcode") |
| Server handler | `server:modules/match/engine/phase/game/phase.go:352-372` (`HandleMessage`) → `Submit` (`:93-133`) |
| Client sender | `client:Application/Match/DuelProtocol.cs:25-37` (`Send`), used by `CastSpellCommandHandler`, `MeditateHandler`, HUD `clear_queue` |

## Payload

| Field | Type | Notes |
|---|---|---|
| `client_seq` | uint64 | positive, strictly increasing per player per match |
| `kind` | string | `cast`, `meditate`, `clear_queue` |
| `spell_id` | string | required for `cast`, must be absent for the other kinds (`omitempty` on both sides) |

```json
{"client_seq":7,"kind":"cast","spell_id":"firebolt"}
{"client_seq":8,"kind":"meditate"}
{"client_seq":9,"kind":"clear_queue"}
```

Parsing is strict: max 1024 bytes, `DisallowUnknownFields`, no trailing JSON (`phase.go:356-367`). Parse failures are dropped with a warning and produce no result; validation failures produce an `action_rejected` on [[op_30_combat_result]]. Player identity is always the message sender.

## Submit-time validation (`Submit`, in order)

| Check | `reason` |
|---|---|
| sender not in `Players` | `not_participant` |
| combat over or sender dead | `combat_ended` |
| `client_seq == 0` | `invalid_client_seq` |
| `client_seq == last accepted` | silently ignored (idempotent retry, no result) |
| `client_seq < last accepted` | `stale_command` |
| unknown `kind` | `unknown_command` |
| `cast`: spell not in sender's `SelectedSpells` | `unknown_spell` |
| `cast`: `mana < mana_cost` | `insufficient_mana` |
| non-cast with `spell_id` | `unexpected_spell_id` |
| sender paralyzed | `paralyzed` |
| `meditate` while poisoned or at full mana | `meditation_unavailable` |

Accepted commands go to `pending[player]` (one slot, latest wins) and are moved to `queued[player]` on the next tick with an `action_queued` result. See [[combat-v2#Commands]] for the tick-time semantics and the second set of rejection reasons.

## Client behaviour

`DuelProtocol.Send` allocates the sequence from `DuelState.Next` (`client:Core/Match/DuelV2.cs:95`) once per intent; a failed socket send does **not** rewind it. Nothing is sent when the protocol was not accepted, the snapshot says `over`, or the tutorial battle is active (routed to `TutorialSession.Battle.Submit` instead).

## Source of truth in code
- `server:modules/match/engine/phase/game/phase.go` — `CombatCommand` struct, `HandleMessage`, `Submit`
- `server:modules/match/engine/phase/game/reasons.go` — submit-time reasons
- `client:Core/Match/DuelV2.cs` — `DuelCommand`, `DuelState.Next`
