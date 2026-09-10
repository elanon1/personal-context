---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-09
verified: 2026-09-09
tags: [hexbane, client, tutorial, onboarding]
sources: ["client:docs/superpowers/specs/2026-09-06-tutorial-design.md", "client:docs/superpowers/plans/2026-09-06-local-tutorial.md", "client:docs/client/tutorial-verification.md"]
---

# Local tutorial (client)

Decision 2026-09-06: basic combat training runs **locally**, before character creation, on the real duel scene; it never creates a Nakama match or sends combat opcodes. Completion is stored on the account through the `tutorial` RPC ([[rpcs]]). Training grants **no reward** (no XP, level, points); the retired `claim_reward` action is rejected by the server. Character development is a separate, independent lesson.

## Code

| Layer | Files |
|---|---|
| Core (no Godot, no network) | `Core/Tutorial/TrainingBattle.cs` (step machine + local clock), `Core/Tutorial/ProgressionLesson.cs` (`ShouldStart`) |
| Application | `Application/Tutorial/TutorialSession.cs`: `TutorialCommand(action)` → RPC `tutorial` with `{action}` (line 64); DTOs `TutorialState{training_completed, reward_claimed, progression_completed, reward}`, `TutorialResponse{success,error,state,character,spells,combat_protocol,ruleset_id,catalog_version}`; flags `Replay`, `ProgressionReplay`; registered as singleton (`ServiceBootstrapper.cs:101`) |
| Game | `Game/ScenesV3/Tutorial/{TutorialScreen,TutorialOverlay,TutorialControls}.cs`, `TutorialScreen.tscn`; HUD targets `ReferenceHud.TutorialControl/TutorialTarget` (`ReferenceHud.cs:215-221`); menu lesson in `CharacterDetail/CharacterDetailScreen.Tutorial.cs` + `.Responsive.cs:RevealTutorialTarget` |
| Dev | `Game/ScenesV3/Dev/TutorialVerification.tscn` |

## Routing

- `SceneManager.GoToDashboard` (`SceneManager.cs:281-287`): if `TutorialSession.State.TrainingCompleted` and no replay flag → `character_creation` (no character) or `dashboard`; otherwise `TutorialScreen`. `GoToCharacterCreation` always opens `TutorialScreen` (line 297-300), which decides.
- `TutorialScreen` starts the arena when `Replay` or training incomplete (`TutorialScreen.cs:43`). On finish it calls `complete_training` unless replaying (line 151), resets `MatchContext`, then routes to `settings` (replay) or `RouteAfterTraining` → `character_creation` when no character, else the menu (lines 49-56). A failed save shows "SAVE PROGRESS / Connection interrupted" with Retry.
- Replays: Settings screen buttons "Replay basic training" (`session.Replay = true`) and "Replay character development" (`session.ProgressionReplay = true`), both loading `TutorialScreen.tscn` (`SettingsScreen.cs:71-82`).
- Back is ignored on `TutorialScreen` (`SceneManager.cs:356, 391`). There is no skip button.

## Training steps (`TrainingStep`, `TrainingBattle.cs:9-13`)

`Welcome → Loadout → Standards → ArrowIntro/ArrowPractice → MirrorIntro/MirrorPractice → MeditationIntro/MeditationPractice → PoisonIncoming → PoisonIntro/PoisonPractice → BarrierIntro/BarrierPractice → FinaleIntro/FinalePractice → Races → Summary → Complete`

- The clock and input run only in practice steps and `PoisonIncoming` (`Paused`, line 39); `PoisonIncoming` is watch-only (line 41): the mentor casts `poison`, the dart flies as a pending impact, poison ticks, then the "You've been poisoned" card explains that poison blocks meditation.
- Spotlight targets per step (lines 43-49): `standards` (both standard slots), `magic_arrow`, `mirror_reflection`, `meditate`, `cleanse`, `barrier`, `loadout`. Meditation steps have no control spotlight; guidance is an animated SVG hand with an upward swipe.
- Practice conditions come from the local model: arrow hit, successful reflection (failure retries without losing), +20 mana from meditation (line 113), cleanse, barrier absorption, finale kill (enemy at 24 HP, line 66). Spell numbers (cost, cast time, values, durations) come from the server catalog delivered by the `tutorial` RPC (`spells` with `effects`), not from constants.
- Race example shown: Human has one extra starter pick (4 vs 3). Other race combat multipliers are inactive in duel v2 and are not presented as working.

## Development lesson (`ProgressionLesson`, `CharacterDetailScreen.Tutorial.cs`)

Starts after the first **match** level-up that leaves points (`ProgressionLesson.ShouldStart`; never at level 1, never from a match-less level change). Steps: Stats tab → allocate → Spellbook → compare/buy → Summary skills. Real allocation/purchase use the existing RPCs and advance only on success; if MP is short the lesson lets the player keep points. `ProgressionReplay` uses isolated demonstration data and never mutates the character (`CharacterDetailScreen.Tutorial.cs:28`). Completion is persisted with `complete_progression` (`CharacterDetailScreen.Tutorial.cs:137`).

## Verification

```sh
dotnet run --project Tests/Tutorial/Tutorial.csproj           # step machine + ShouldStart cases
python3 Tests/Tutorial/live_rpc.py                            # local Nakama 127.0.0.1:7350, disposable device account
HEXBANE_IGNORE_ENV_FILE=1 DEV_AUTO_LOGIN=false NAKAMA_HOST=127.0.0.1 NAKAMA_PORT=7350 \
TUTORIAL_SIZE=phone TUTORIAL_INPUT=touch \
  /Applications/Godot.app/Contents/MacOS/Godot --path . --scene res://Game/ScenesV3/Dev/TutorialVerification.tscn
```
`TUTORIAL_SIZE` = `phone` (1360×612) | `wide` (1560×720) | `pc` (1920×1080) | `tablet` (1024×768), default 1360×768; `TUTORIAL_INPUT=touch` uses synthetic touch; `TUTORIAL_CAPTURE=1` saves `.standards.png`, `.poison.png`, `.poisoned.png` (`TutorialVerification.cs:4-24, 80-140`). The scene creates a disposable local account.

Recorded 2026-09-07: C# build OK, Tests/Tutorial pass, Go tests pass, `live_rpc.py` passes, Godot acceptance at 1360×612 passes. Not tested: physical phone, a real multiplayer level-up end to end. Godot still reports ObjectDB leaks at forced shutdown.

## Plan items not implemented

From the design/plan: nothing from the approved local variant is outstanding. The earlier "one-time server reward" and `claim_reward` in the plan text were superseded by the no-reward decision (server rejects `claim_reward`; the client `TutorialReward` DTO still exists for the `reward` field).

## Source of truth in code
- `client:Core/Tutorial/TrainingBattle.cs` — steps, pausing, targets, exercise rules
- `client:Core/Tutorial/ProgressionLesson.cs` — development-lesson trigger
- `client:Application/Tutorial/TutorialSession.cs` — `tutorial` RPC actions and DTOs
- `client:Game/ScenesV3/Tutorial/TutorialScreen.cs` — arena controller and routing
- `client:Game/Autoloads/SceneManager.cs` — gating on `TrainingCompleted`
- `client:Game/ScenesV3/Settings/SettingsScreen.cs` — replay entry points
- `client:Tests/Tutorial/*`, `client:Game/ScenesV3/Dev/TutorialVerification.cs` — verification

## Shared spell presentation repair (2026-09-08)

The local model now emits the same `cast_released` kind as duel presentation (the retired local `spell_release` was ignored). Cast start, snapshot action, release, damage/reflection and impact share a stable action id. Mentor Firebolt emits a final `spell_impact`, including the reflected target, so the shared projectile resolves rather than fading unresolved. Status application/removal carries instance id and ownership; poison damage references its actual status instance. Damage with absorption precedes shield depletion, matching server event ordering. Tutorial recasts replace the prior same-kind status to match its refreshed local deadline.

`TutorialScreen` keeps the resolved arena visible for one second before the next explanation covers it. The lesson clock and input are already paused during this feedback interval; no combat result is delayed. Persistent poison is restored by the shared manager from status events/snapshots. No tutorial-specific renderer or duplicate spell effects are introduced.

Cleanse now uses the shared `cast_2h` preset gesture: 24 pre-release frames for each race. The previous `area_2h_02` had release frame 3–4 on most races and stretched those few frames across the cast, then compressed the remaining frames into recovery. The golden cleansing sweep, cast duration and gameplay remain unchanged.

Verification: `Tests/Tutorial` additionally checks the canonical event contract, stable action identities, reflected impact and status refresh; `VerifyCleanse.tscn` checks motion, recovery and field expiry on all six races. `TUTORIAL_VFX_ONLY=1` makes `TutorialVerification.tscn` stop at Summary, before account completion/character creation tests; it exercises real controls and asserts shared Arrow/Firebolt release/impact, visible Poison/Cleanse and the unobscured feedback interval. Finale verification uses Firebolt, since the current one-damage Magic Arrow cannot remove 24 HP with the initial 100 mana alone. Evidence: `verification/tutorial-vfx/`.

Validation 2026-09-08: build 0 errors / 9 existing warnings; Core tutorial tests PASS; Cleanse GPU-independent motion/lifecycle verifier 18/18; real tutorial GPU/touch acceptance at 1360×612 PASS through Summary using local Nakama authentication/catalog. Shared VFX assertions included. Physical Android untested.

## Combat desktop/mobile profiles (2026-09-09)

See [[combat-ui-profiles]] for the shared profile mechanism, configurable keyboard actions, preview scenes and validation. Desktop uses the bottom action bar and visible keycaps; mobile keeps touch rails. The tutorial follows the chosen profile and respects configured keys and target gates.
