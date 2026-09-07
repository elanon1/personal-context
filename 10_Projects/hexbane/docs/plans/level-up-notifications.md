---
type: project
project: Hexbane
area: plans
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, plans, notifications, progression]
sources: ["server:docs/plans/level-up-notifications.md"]
---

# Plan: persistent level-up notifications (NOT implemented as of 2026-09-07)

## Current state (verified)
- Level-up detection: `Character.AddExp` (`server:modules/character/character.go:117-135`) returns
  `*progression.LevelUpResult` (`NewLevel, LevelsGained, StatPointsGained, MagicPointsGained, NewSpellSlots`).
- Called from the game-over phase (`server:modules/match/engine/phase/gameover/phase.go:224`); the result is
  written to the DB (`:257`) and sent per player inside the opcode **50** `PlayerMatchResult.level_up`
  (`phase.go:28,49-56,285-292`) as `{"previous_level","new_level","stat_points_gained","magic_points_gained","new_spell_slots"}`.
- `NotificationCodeLevelUp = 4` exists (`server:modules/notifications/types.go:20`) but no code sends it
  (`grep NotificationCodeLevelUp` hits only the constant). The client already has `NotificationType.LevelUp = 4`.
- Blocker unchanged: phases receive no `runtime.NakamaModule`; `MatchState`
  (`server:modules/match/engine/state/state.go:35-59`) has no `Nk` field.

## Steps
1. Add `Nk runtime.NakamaModule` to `MatchState` and a parameter to `CreateInitialMatchState(mode, nk)`
   (`state.go:61-77`); pass `nk` from `normal_match/init.go:71-80` and `ai_match/init.go:53-64`.
2. Add `SendLevelUpNotification(ctx, nk, logger, userID, characterName, *progression.LevelUpResult)` to
   `server:modules/notifications/rpc.go` — subject `Level Up!`, code 4, persistent, content
   `{"type":"level_up","character_name","new_level","levels_gained","stat_points_gained","magic_points_gained","new_spell_slots","message"}`.
3. In `gameover/phase.go` after the DB update (`:257`) and inside `if levelUpResult != nil` (`:262`),
   call the helper with `ms.Nk`; log and continue on error. Bots return earlier (`:201`) so they are excluded.
4. Client: handle code 4 in `NotificationModule` (currently only listens generically).

## Notes
- Non-blocking: notification failure must not affect match result processing.
- The old plan's line numbers (`character.go:106`, `phase.go:137`, `ai_match/init.go` `spellSlots = 4`) are stale; those above are current.
- Do not reuse code 1: `remove_friend` already sends code 1 as a generic "friends" notification (see [[notifications]]).

## Source of truth in code
- `server:modules/match/engine/phase/gameover/phase.go` — where level-up is known and would be notified.
- `server:modules/match/engine/state/state.go` — match state constructor to extend.
- `server:modules/notifications/rpc.go`, `types.go` — helper pattern and code constants.
- `server:modules/progression/xp.go` — `LevelUpResult`.
