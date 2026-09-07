---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, tutorial, onboarding]
sources: ["server:docs/tutorial.md", "client:docs/client/tutorial-verification.md"]
---

# Tutorial (server side)

The tutorial is played **locally in the Godot client**; the server only persists account-scoped
completion flags. No Nakama match is created (`server:modules/character/tutorial.go:36`).

## RPC `tutorial` (`server:modules/character/init.go:45`, handler `tutorial.go:106`)

Request `{"action":"status"|"complete_training"|"complete_progression"}`; empty payload = `status`.

Unlike the other RPCs this one raises gRPC errors: `Authentication required` (code 16) and
`Invalid tutorial request` (code 3) (`tutorial.go:107-118`). An unknown action or a failed
transaction returns HTTP 200 with `{"success":false,"error":"Could not save tutorial progress. Please retry.",…}`.

Response (`tutorial.go:27-34`):
```json
{"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2",
 "success":true,
 "state":{"training_completed":false,"reward_claimed":false,"progression_completed":false,
          "reward":{"level":0,"stat_points":0,"magic_points":0,"bonus_magic_points":0,"meditation":0}},
 "character": <Character>|absent,
 "spells": [<full spell_system.Spell>…]   // only for action=status
}
```
- `character` is the caller's character (`ToResponse`, see [[rpcs]]) when one exists; the basic lesson
  runs before creation, so it may be absent.
- `spells` is the whole 14-spell catalog, times in **seconds**, with `effects[]` — the client builds
  the local training battle from it (`client:Application/Tutorial/TutorialSession.cs:90-94`).
- `state.reward` is the historical receipt (`account_tutorials.reward_receipt`); nothing writes new rewards.

### Actions (`tutorial.go:37-104`)
| Action | Effect |
|---|---|
| `status` | Upserts the `account_tutorials` row, returns flags + character + catalog. |
| `complete_training` | Sets `training_completed = true`. Grants **nothing** (no XP, level, MP, stat points, skills). |
| `complete_progression` | Requires a character with `level ≥ 2`, and that the player has actually spent something: rejected while `unspent_stat_points > 0` with base stats still at 400, or while `magic_points_spent == 0` and the cheapest non-standard spell (5 MP) is affordable. On success sets `progression_completed = true` **and** `characters.tutorial_completed = TRUE`. Failure → `success:false`, error `Practice progression after earning and spending your first points` (logged; client sees the generic retry text). |
| `claim_reward` (legacy) | Unknown action → rejected before any DB access. |

Row is locked `FOR UPDATE` for the whole operation; the character row too during `complete_progression`.

## RPC `set_tutorial_completed` (retired)
`server:modules/character/rpc.go:153` always returns
`{"character":null,"message":"Use tutorial progression; tutorial state cannot be reset","success":false}`.
The client wrapper `CharacterService.SetTutorialCompleted` has no callers.

## Storage
- `account_tutorials(user_id PK → users.id, training_completed, reward_claimed, progression_completed, reward_receipt JSONB, updated_at)` — migration `server:db/migrations/000003_local_tutorial.up.sql`. Existing characters were back-filled as `training_completed = reward_claimed = TRUE`, `progression_completed = tutorial_completed`.
- `characters.tutorial_completed` (migration 000001:53) is now only set by `complete_progression`; it is returned in every `<Character>` but nothing on the client gates on it.

## Client flow (verified in code)
- `TutorialSession` (`client:Application/Tutorial/TutorialSession.cs`) is a DI singleton; `Operation(action)` dispatches `TutorialCommand` → RPC `tutorial`; rejects unsupported catalog versions.
- `SceneManager` routes to `TutorialScreen.tscn` unless `State.TrainingCompleted` and no replay is requested (`client:Game/Autoloads/SceneManager.cs:283-299`); Settings offers replays (`SettingsScreen.cs:73-82`).
- `TutorialScreen` calls `status` on load and `complete_training` at the end (`Game/ScenesV3/Tutorial/TutorialScreen.cs:39,151`) unless replaying.
- `CharacterDetailScreen.Tutorial.cs` runs the progression lesson after the first level-up and calls `complete_progression` (`:137`).
- Local training uses match id `"local-training"`; match managers skip rejoin for it.
- Verification artefacts: `client:Tests/Tutorial/` (C# project + `live_rpc.py` against a local Nakama), acceptance scene `client:Game/ScenesV3/Dev/TutorialVerification.tscn` (env `TUTORIAL_SIZE=phone|wide|pc|tablet`, `TUTORIAL_INPUT=touch`, creates a disposable device-login account). Step order as of 2026-09-07: loadout → standards → … → meditation → incoming poison (`tutorial-verification.md`, not re-verified here).

## Source of truth in code
- `server:modules/character/tutorial.go` — actions, eligibility rule, response shape.
- `server:modules/character/rpc.go:152-155` — retired `set_tutorial_completed`.
- `server:db/migrations/000003_local_tutorial.up.sql` — `account_tutorials`.
- `client:Application/Tutorial/TutorialSession.cs` — DTOs and command.
- `client:Game/Autoloads/SceneManager.cs`, `client:Game/ScenesV3/Tutorial/TutorialScreen.cs`, `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.Tutorial.cs` — when each action is sent.
- `client:Core/Tutorial/TrainingBattle.cs`, `ProgressionLesson.cs` — local lesson logic.
