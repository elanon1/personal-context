---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, game-over]
sources: ["client:docs/opcodes/op_50_game_over.md", "server:docs/opcodes/op_50_game_over.md"]
---

# Opcode 50 — GameOver

| | |
|---|---|
| Server const / client enum | `OpGameOver` / `GAMEOVER` |
| Direction | Server → client, unicast per player (bots skipped: no presence) |
| Phase | `combat_end` |
| Sender | `server:modules/match/engine/phase/gameover/phase.go:133-179` (`Tick`), `sendPlayerResult` `:335-350` |
| Client handler | `client:Application/Match/Incoming/Gameover/GameoverHandler.cs` |

## When

On the first tick of `combat_end`, right after the final [[op_32_combat_snapshot]] (`over:true`) and the `match_ended` event. The server persists XP/skills/record first, sends one personalized message per player, then sets `TerminateMatch`, so the match ends on the following tick regardless of [[op_199_quit_game]]. If `determineOutcomes` finds fewer than two players the message is not sent at all (`:140-145`).

## Payload (`PlayerMatchResult`, `phase.go:23-87`)

| Field | Type | Notes |
|---|---|---|
| `outcome` | string | `victory`, `defeat`, `draw` |
| `reason` | string | `defeated`, `timeout`, `draw` |
| `opponent` | `{user_id, username}` | |
| `xp` | `{gained, total_experience, experience_to_next, current_level, is_first_win_bonus}` | ints + bool |
| `level_up` | `{previous_level, new_level, stat_points_gained, magic_points_gained, new_spell_slots}` | omitted when no level-up |
| `skill_gains` | `{meditation, spell_resistance, magery}` each `{previous, current, gained}` (float64) | `gained` is 0 in duel_v2: no code path calls `TriggerMageryGain`/`TriggerSpellResistanceGain` (see report) |
| `stats` | `{level, strength, intelligence, dexterity, spell_slots, magic_points, unspent_stat_points}` | post-match character row |
| `record` | `{wins, losses}` | |

Bots (and any player whose character row cannot be loaded) get only `outcome`, `reason`, `opponent` with zero-valued structs (`:202-211`).

```json
{"outcome":"victory","reason":"defeated","opponent":{"user_id":"0000","username":"Bot"},
 "xp":{"gained":50,"total_experience":150,"experience_to_next":100,"current_level":2,"is_first_win_bonus":true},
 "level_up":{"previous_level":1,"new_level":2,"stat_points_gained":5,"magic_points_gained":2,"new_spell_slots":3},
 "skill_gains":{"meditation":{"previous":10,"current":10,"gained":0},"spell_resistance":{"previous":10,"current":10,"gained":0},"magery":{"previous":10,"current":10,"gained":0}},
 "stats":{"level":2,"strength":120,"intelligence":200,"dexterity":80,"spell_slots":3,"magic_points":7,"unspent_stat_points":5},
 "record":{"wins":1,"losses":0}}
```

Outcome rules (`determineOutcomes`, `:105-131`): exactly one player with HP > 0 → `defeated`; otherwise `timeout` if the combat phase ended by clock, else `draw` (both dead in the same impact batch, `DeferDeath`).

## Client behaviour

`ChangeStatus(MatchEnded)`, copies `stats`, `xp`, `record`, `skill_gains.*.current` into the cached `GameContext.Character`, stores `GameEvents.LastGameOverData`, raises `onGameOver(message)`. The DTO mirrors every field (`GameoverMessage.cs`).

## Source of truth in code
- `server:modules/match/engine/phase/gameover/phase.go` — structs, outcome logic, persistence
- `client:Application/Match/Incoming/Gameover/GameoverMessage.cs` — DTO
