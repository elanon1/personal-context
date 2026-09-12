---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
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

After the final [[op_32_combat_snapshot]] (`over:true`) and `match_ended`, the server atomically stores XP/skills/record plus the personalized opcode50 payload in the existing `character_match_rewards.result` JSON. Transient persistence failure keeps settlement pending and retries; successful replay uses the frozen payload without recalculating or paying twice. Bots are skipped. Once personalized results are sent, the match terminates.

## Payload (`PlayerMatchResult`, `phase.go:23-87`)

| Field | Type | Notes |
|---|---|---|
| `outcome` | string | `victory`, `defeat`, `draw` |
| `reason` | string | `defeated`, `timeout`, `draw` |
| `opponent` | `{user_id, username}` | |
| `xp` | `{gained, total_experience, experience_to_next, current_level, is_first_win_bonus}` | ints + bool |
| `level_up` | `{previous_level, new_level, stat_points_gained, magic_points_gained, new_spell_slots}` | omitted when no level-up |
| `skill_gains` | `{meditation, spell_resistance, magery}` each `{previous, current, gained}` (float64) | frozen before/after skill values from settlement |
| `stats` | `{level, strength, intelligence, dexterity, spell_slots, magic_points, unspent_stat_points}` | post-match character row |
| `record` | `{wins, losses}` | |

Fallback opponents use their stable persona UUID/name in `opponent`; there is no wire `is_bot`. Explicit training retains its separate identity. Current XP is governed by [[progression]] (120 victory /70 defeat or draw, no first-win bonus); the previous 50XP sample was obsolete. Current skills are applied through settlement; the old statement that all skill gains remain zero is obsolete.

### Recovery

Authenticated `get_match_result {match_id}` returns `{success:true,data:<the personalized opcode50 object>}` only for the caller’s character receipt. Unknown match, another player’s match and historical receipts without a saved wire payload return NotFound. It does not infer an old outcome from current progression. New normal/ranked/training results store payloads without a new ledger/migration.

Normal client de-duplicates live/recovered result delivery. On reconnect it checks the receipt before joining a possibly ended match; an `over:true` snapshot without opcode50 starts bounded retries (1s initially, then3s, 30 attempts), fenced by match/account. Failure surfaces a reconnect retry message. This recovers within the current running client; there is no persisted app-restart match history browser.

Outcome rules (`determineOutcomes`, `:105-131`): exactly one player with HP > 0 → `defeated`; otherwise `timeout` if the combat phase ended by clock, else `draw` (both dead in the same impact batch, `DeferDeath`).

## Client behaviour

`ChangeStatus(MatchEnded)`, copies `stats`, `xp`, `record`, `skill_gains.*.current` into the cached `GameContext.Character`, stores `GameEvents.LastGameOverData`, raises `onGameOver(message)`. The DTO mirrors every field (`GameoverMessage.cs`).

## Source of truth in code
- `server:modules/match/engine/phase/gameover/phase.go` — structs, outcome logic, persistence
- `client:Application/Match/Incoming/Gameover/GameoverMessage.cs` — DTO

## Duel ending presentation (2026-09-12)

Client presentation now retains the arena until Continue, when an active DuelConclusion exists. Character cache/result dispatch are still immediate and opcode 50 is unchanged. Terminal actor death follows HP or the explicit defeated outcome; timeout does not create a death. See [[duel-ending]].
