---
type: project
project: Hexbane
area: server
status: review
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, audit, cleanup, dead-code]
sources: ["server:modules", "server:cmd", "server:vendor/github.com/heroiclabs/nakama-common/runtime/runtime.go"]
---

# Server unused-code and cleanup report — 2026-09-08

## User decision and implementation update — 2026-09-08

**This section supersedes deletion recommendations below. The original inventory is a historical pre-cleanup snapshot; its line numbers and 61-count are not current remaining-work counts.**

- Approved and implemented: unused character/spellbook/catalog/race/player/debug/story-local helpers; unused notification helpers and DTOs; commented social implementations; old `CastInterruptions` buffer. Notification list/delete RPCs and the real social notification remain.
- `MatchLog` is explicitly retained, including existing writes and resets. `EffectRemovalEvents` was not removed.
- Combat formulas, resistances, regeneration, stats, racial mechanics and skills are **not approved for deletion**. They and their tests were restored into live combat after the user approved “przywroc”; see [[combat-stat-rules]]. Sections B/C are withdrawn as deletion recommendations.
- `get_progression` fixed using the shared progression helpers. New regression test failed on the old slot ladder and passes after the fix; next-level XP is clamped to zero. API shape unchanged.
- Tests for paralysis now assert the real lifecycle `cast_interrupted` event (player/action/spell/reason), no mana refund and preserved MatchLog entries instead of the removed buffer.
- Removed 33 declarations from the original snapshot (including the replaced `pow`); added the directly tested response builder. Final `go test ./...`, `go vet ./...`, `git diff --check`: PASS. Race tests for match/spell_system/simulator passed after buffer removal.
- Story feature retirement, registered RPC retirement, database changes, test-only helper removal and whole-module deletion were not performed.

## Scope and conclusion

Audit of the **current working tree**, including modified and untracked files, as explicitly accepted by the user. Branch `feat/spell-system-redesign`, HEAD `017ed09`; HEAD alone does not reproduce this snapshot. No application code was changed during the original audit; see the implementation update above.

Inspected 451 production function/method declarations under `modules/` and `cmd/`. Found **61 declarations without production callers/references after framework exclusions and manual name-collision checks**: 48 also have no test references, 13 have test references. These are cleanup candidates, not 61 automatic deletions. Appendix A lists each one. Additional transitive dead code and unreachable phase methods are described separately, not added to that total.

The main source of clutter is coexistence of the old stat/skill combat model and the active fixed-value `duel_v2` model, plus unfinished story/social/notification scaffolding. The current contract already documents 200 HP / 100 mana, fixed damage and no skill gains; cleanup should preserve that behavior.

## Method and limitations

- Parsed all Go files under the two roots using Go AST; counted identifier references separately in production and tests, excluding function declarations, comments and string literals. This avoids counting commented-out calls and logger strings as usage.
- Name-based counting is conservative: unrelated methods/fields with the same name can hide unused code. Manually resolved `Match.Label`, `PlayerState.TriggerMageryGain`, `SkillGains.Get`, and `SpellRegistry.Add` and traced important dead dependency groups.
- Reviewed `modules/main.go`, module registries, the vendored `runtime.Match` interface and effect-handler dispatch. Preserved callbacks, registered RPCs and interface methods. No reflection-based lookup of listed helper names was found in application code.
- This is **not** a complete type-aware whole-program reachability proof. `staticcheck`, `deadcode` and `golangci-lint` were not installed. An attempted standard-library type-aware analyzer could not start because this local Go environment reported `go/importer` and `go/types` as unavailable. AST analysis and manual source tracing were used instead.
- Client source and live RPC traffic were not audited in this session. A registered RPC cannot be declared unused based on absence of Go callers. Existing client findings in the vault are contextual evidence, not freshly verified client findings.
- Vendor, generated/build artifacts and deleted working-tree files are excluded from deletion counts. Infrastructure assets are not classified as unused merely because Go does not reference them.

## A. First cleanup: remove isolated dead helpers

All symbols in these groups have no production or test reference unless explicitly noted. Deleting the functions is low risk within this repository; delete imports/constants that become unused, but keep surrounding live APIs.

| Group | Remove | Reason / boundary |
|---|---|---|
| Character | `Character.SpendMagicPoints`, `FindByCharacterName`, `validateSpellsExist`, `Character.UpdateSkills` | No callers. Spell purchase and match-result persistence have separate live paths. Do not delete `LearnSpell`, `UpdateMatchResult` or creation validation. The comment claiming internal callers for `validateSpellsExist` is stale. |
| Spellbook | `ValidateSpellExists`, `ValidateSpellNotLearned` | Unused duplicate validation. Keep `ValidateCharacterExists` and the live ownership/learning checks in RPC/DB code. |
| Catalog | `GetSpells`, `SpellRegistry.GetByLevelRequirement`, `SpellRegistry.LoadFromFile`, `SpellRegistry.Add` | Live catalog loads through `LoadFromDirectory`. `GetSpells` ignores its school/limit arguments; `Add` bypasses catalog validation. After removing `LoadFromFile`, simplify its list-form parsing branch only after checking remaining tests/callers. |
| Race | `FindById` | Live lookup uses the race registry; retain startup `FindAll`/scan helpers. |
| Common | Entire `modules/common/utils.go` | `RandString` and its private RNG/alphabet have no consumer. Keep `common/types.go`. |
| Debug | Entire `phase/game/utils.go`, entire `phase/lobby/debug.go` | `DebugPlayerEffects` and `DebugForcePickSpellsForAllPlayers` are uncalled. Also remove the commented debug invocation at `lobby/phase.go:333`. |
| Player state | `HaveMana`, `GetSpellSlots`, `HasSpell`, `SetCastingSpell`, `GetSkillGains` | No callers. Keep live casting in `actions.go`, `GetSpell`, `GetCastingSpell`, `UnsetCastingSpell`, and direct state used by the duel. |
| Snapshot | `PlayerSnapshot.IsEqual` | No caller. It also compares effect keys rather than full instance/state values; do not repurpose it as a duel-v2 delta comparator. Keep `PlayerSnapshot` and `GetSnapshot`. |
| Progression | `CalculateTotalMagicPoints`, `CalculateLevelFromXP`, `GetSpellSlotProgress` | Unused convenience APIs. Remove `SpellSlotProgress` with its helper if it remains unreferenced. Keep active XP/MP/slot formulas. |
| Match metadata | `ai_match.Match.Label`, `create_character.Match.Label` | Neither is part of the vendored `runtime.Match` interface and neither has callers. A `Label` field elsewhere is an unrelated identifier. |
| Story local helpers | `remainingFields`, `toStringSlice`, `toStringIntMap`, `CharacterProgress.IsComplete` | Referenced only by comments. Can be removed independently of the registered story feature decision. |

## B. Remove the dormant combat-formula subsystem, keeping effective stats

`modules/combat/damage.go` and `modules/combat/resistance.go` have no live consumer. Their helpers reference each other, so simple zero-reference searches understate the dead subsystem.

Recommended deletion:

- All of `damage.go`: `CalculateScaledDamage`, `GetStatResistance`, `CalculateHealAmount`, `EffectDamageInstances`, `DamageContext`, damage-type declarations.
- All of `resistance.go`: `CalculateTotalResistance`, `ApplyResistance`, `ResistanceContext`.
- From `stats.go`: `CalculateMaxHealth`, `CalculateMaxMana`, `CalculateCastingTimeBonus`, `CalculateDodgeChance`, `CalculateManaRegenBonus`, `CalculatePassiveManaRegen`, `CalculatePassiveHealthRegen`, and their now-unused constants.
- Remove obsolete formula tests in `damage_test.go` and the corresponding cases in `stats_test.go` **with the obsolete implementation**, not before it.

**Keep `CalculateEffectiveStats`** (`stats.go:107`): called by character creation, allocation and recalculation at `character/character.go:81,163,190`. Do not delete the whole `combat` package blindly; moving this one live helper to the character domain is an optional later refactor.

Evidence of the replacement: `core/player_setup.go:33-34` fixes HP/mana; `effect_handlers/damage.go:78` returns the catalog value; `state/actions.go` implements the live resource/action rules. `EffectDamageInstances` even encodes obsolete pulse semantics (duration/interval + 1), unlike the current queue.

After this deletion, `skills.GetSkillEffectiveness` and `skills.GetSkillReduction` lose their only production references and can go too. They are transitive candidates, not included in the 61 direct-reference total.

## C. Skill-gain scaffolding: remove behavior code in a separate change

Direct dead entry points: `PlayerState.TakeDamageWithSkillGain`, `TakeDamageWithSkillGainAndEvents`, `TriggerMageryGain`; `skills.TriggerMeditationGain`, `NewSkillGains`, `SkillGains.Get`.

The remaining dependency chain is also dormant in normal gameplay:

`PlayerState.TriggerMageryGain / TakeDamageWithSkillGain*` → `skills.TriggerMageryGain / TriggerSpellResistanceGain` → `TryGainSkill` → `SkillGainChance` → `SkillGains.Add`.

`BuildPlayerState` sets `SkillGains: nil` (`core/player_setup.go:77`); no normal production initializer was found that creates the tracker. Game-over `HasGains` / `ApplySkillGains` / `CalculateNewSkillValue` is nevertheless referenced and should be removed as an explicit behavior-path cleanup, not mislabeled as zero-reference functions.

Recommended boundary: remove gain triggers, random-gain implementation and the unused match tracker; preserve stored skill values and existing `skill_gains` response shape until a separate protocol/data decision. Do not drop DB columns, race traits or character statistics just because they do not influence combat. They are persisted/returned and used by menu/creation code.

## D. Legacy event buffers: work that executes but has no output consumer

- `MatchLog` is appended by damage/heal handlers, forwarded through `EffectContext`, then cleared by `GamePhaseState.Advance` (`phase.go:204`) and match end (`:330`); the simulator also clears it. No production serializer/reader was found. Remove the type/field, construction, forwarding, appends and resets as one change.
- `CastInterruptions` is appended by `ParalyzeHandler` (`paralyze.go:44`) and cleared at `phase.go:202`. Live `cast_interrupted` events are emitted separately by `spell_effects.ApplyEffect` (`engine.go:27-46`). Remove the old buffer path and migrate assertions in `paralyze_test.go` to the lifecycle event sink before deletion.
- `EffectRemovalEvents` has fallback writes when no sink is attached (`events.go:50`, `queue.go:131`) and is cleared at `phase.go:203`. Investigate/remodel the fallback and tests together. **Keep `EffectRemovalEvent` where the live damage-to-queue path uses it; do not delete all removal-event code.**
- Remove unused removal-reason constants `EffectRemovalReasonBroken`, `EffectRemovalReasonConsumed`, `EffectRemovalReasonCured` if searches still show definitions only. Keep active `Expired` and `Depleted` constants.

The replacement is the active effect queue/lifecycle sink and protocol events 30/31/32. This cleanup must preserve shield depletion, paralysis interruption and event ordering.

## E. Notifications: unused helpers and a literal example stub

Remove the six uncalled `Send*Notification` helpers listed in Appendix A and their misleading startup log entries in `notifications/init.go:29-35`. Social code sends its live notification directly through `nk.NotificationSend` (`social/rpc.go:344`), so keep that path.

Delete `CreateNotificationContent` (`types.go:136`): despite its name/comment it returns `content.Message`, not JSON. This is example scaffolding, not a useful serializer. Review/remove the associated unused send-request/response and notification payload DTOs after reference checking. Keep DTOs used by list/delete RPCs.

`notifications_list` and `notifications_delete` are registered and therefore **not dead functions**. The existing vault note says the client uses native SDK calls; removing those RPCs is an API-surface decision requiring a current client scan. Do not infer authorization to remove the entire module from this audit.

## F. Story feature: registered but incomplete, not simply unused

`create_character_match_story` calls `MatchCreate("create_character")` (`create_character/init.go:31`), while startup registers `"v2_create_character"` (`:25`). Its RPC path is broken by this name mismatch. `CreateCharacterState.HandleMessage` unmarshals JSON but all progress mutation/completion code is commented out. `DoneState` has no live construction; its `Tick` and `HandleMessage` are additional unreachable methods concealed by shared method names.

Recommended product decision: retire this unfinished story creation feature if current direct character creation plus the local tutorial are the intended product. Then remove the `endless_story` registration/import from `modules/main.go`, `modules/endless_story/**`, and update RPC/story-opcode docs and client references together. Alternatively complete it as a separate feature. A naming fix alone does not make the feature complete.

Do not remove the registered match callback methods individually while retaining the module: they satisfy `runtime.Match`.

## G. Test-only APIs: decide deliberately

13 entries in Appendix A have test references. Their treatments differ:

- Legacy combat formula tests: remove alongside the obsolete formulas (section B).
- `PlayerState.TakeDamage`, `AddMana`, `ReduceMana`: consider migrating tests to the live damage/resource APIs, preserving meaningful health/mana boundary and concurrency assertions. Delete wrappers after migration.
- `PlayerState.AddEffect`: used as test setup, while production uses `UpsertEffect` / the queue. Can be retained as an intentional helper or migrated carefully; do not discard effect-lifecycle coverage.
- `race.Registry.Set`: used by tests across package boundaries. Keep for now; moving it blindly into the race package's `_test.go` breaks importing test packages. Alternative: a deliberate fixture-loading API.
- `NewGameOverPhaseState`: test-only default constructor; consolidate the test around `NewGameOverPhaseStateWithReason` or retain for convenience.
- `ProcessLevelUp`: test-only wrapper; live code uses `ProcessLevelUpWithSlotBonus`. Migrate the test to the live function with zero bonus before removing the wrapper.

## H. Other cleanup and one active defect

1. **Fix `RpcGetProgression`, do not remove it.** `character/rpc.go:341-357` duplicates XP calculation and hardcodes old slot levels `[4,6,10,15,20,25]`. The active ladder is `[4,8,12]`. Replace the inline rules with existing progression helpers, then delete the private `pow` (`:389`). `pow` currently has a caller and is not a dead function. The XP expression currently matches the next-level exponent for ordinary valid levels; the confirmed divergence is the slot ladder, while duplicated XP logic is maintenance debt. Add regression cases around levels 4, 6, 8, 12 and the cap.
2. Remove commented-out implementations in `social/rpc.go`, `social/db.go`, `endless_story/init.go`, `phase_create_character.go` and the lobby debug call. Retain explanations of current behavior. Version control is the archive.
3. Remove empty `if ctx.Caster != nil { /* fixed values */ }` blocks and stale skill/modifier comments in damage/heal handlers. `DamageHandler.calculateDamage` is called but currently just casts the catalog value: inline it if that improves clarity.
4. Fix misleading comments: player setup says it calculates HP/mana from stats; the player-state block describes nonexistent `InterruptCast` above `UnsetCastingSpell`; social `FindByName` has a playstyle-copy comment. Remove unused `PhaseStats` after confirming no client contract relies on that value.
5. `modules/social/README.md` and `modules/notifications/README.md` still exist despite the vault-only rule. Compare for unique content, preserve anything current in the corresponding vault notes, then remove those repo docs. Keep `AGENTS.md`, `CLAUDE.md` as agent instructions and `docs/README.md` as the allowed pointer.
6. Helm/Argo and balance-output freshness are separate infra/tooling review items already recorded in the vault. They are not proven dead by this Go audit; no infrastructure deletion is recommended solely from this scan.

## Suggested execution order and acceptance criteria

| Batch | Scope | Validation |
|---|---|---|
| 1 | Isolated unused helpers, debug files, commented code, example notification serializer | Focused package tests, then `go test ./...` and `go vet ./...`; no RPC/schema changes. |
| 2 | Dormant combat formulas and obsolete tests | Preserve effective-stat tests; run character/race tests and full suite. Verify duel constants unchanged. |
| 3 | Match-log/interruption buffers and skill-gain plumbing | Adapt tests to live lifecycle events; full suite and `go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim`; run the existing simulator and runtime scenarios where environment is available. |
| 4 | Active progression RPC defect | Regression tests, RPC response check, update progression/protocol notes. |
| 5 | Story/RPC retirement and persisted data simplification, if desired | Current client-reference audit, explicit API/data scope, integration smoke tests and corresponding vault updates. Do not combine with mechanical deletion. |

For each batch, remove newly orphaned imports/types/constants and rerun reference checks. Do not add tests asserting that a function name is absent. Do not rewrite migrations or drop persisted fields during the mechanical cleanup. Keep commits small enough to review and revert independently.

## Verification performed in this session

- `go test ./...`: PASS (exit 0; reported package results were cached).
- `go vet ./...`: PASS (exit 0).
- Local Go: `go1.24.5 darwin/arm64`, whereas repository guidelines request 1.24.3. No Linux plugin build, live Nakama/DB run or race test was performed for this read-only audit.
- No application files deleted/edited; this report, its index entry and the session journal are the deliverables.

## Appendix A — exact direct-reference candidates

`tests` is the number of identifier references, not the number of test cases. Line numbers refer to this working-tree snapshot. Zero production references are within the audited server source scope; interface callbacks have been excluded. The recommendations above override any interpretation that every row should be deleted.

| Location | Function / method | Test references |
|---|---|---:|
| `server:modules/character/character.go:202` | `Character.SpendMagicPoints` | 0 |
| `server:modules/character/db.go:162` | `FindByCharacterName` | 0 |
| `server:modules/character/db.go:297` | `validateSpellsExist` | 0 |
| `server:modules/character/db.go:318` | `Character.UpdateSkills` | 0 |
| `server:modules/combat/damage.go:140` | `CalculateHealAmount` | 0 |
| `server:modules/combat/damage.go:155` | `EffectDamageInstances` | 1 |
| `server:modules/combat/damage.go:52` | `CalculateScaledDamage` | 5 |
| `server:modules/combat/resistance.go:14` | `CalculateTotalResistance` | 0 |
| `server:modules/combat/resistance.go:41` | `ApplyResistance` | 0 |
| `server:modules/combat/stats.go:102` | `CalculatePassiveHealthRegen` | 0 |
| `server:modules/combat/stats.go:33` | `CalculateMaxHealth` | 3 |
| `server:modules/combat/stats.go:42` | `CalculateMaxMana` | 0 |
| `server:modules/combat/stats.go:51` | `CalculateCastingTimeBonus` | 1 |
| `server:modules/combat/stats.go:67` | `CalculateDodgeChance` | 5 |
| `server:modules/combat/stats.go:87` | `CalculateManaRegenBonus` | 1 |
| `server:modules/combat/stats.go:93` | `CalculatePassiveManaRegen` | 0 |
| `server:modules/common/utils.go:12` | `RandString` | 0 |
| `server:modules/endless_story/create_character/init.go:56` | `Match.Label` | 0 |
| `server:modules/endless_story/create_character/phase_create_character.go:102` | `remainingFields` | 0 |
| `server:modules/endless_story/create_character/phase_create_character.go:122` | `toStringSlice` | 0 |
| `server:modules/endless_story/create_character/phase_create_character.go:139` | `toStringIntMap` | 0 |
| `server:modules/endless_story/create_character/types.go:47` | `CharacterProgress.IsComplete` | 0 |
| `server:modules/match/ai_match/init.go:66` | `Match.Label` | 0 |
| `server:modules/match/engine/phase/game/utils.go:9` | `DebugPlayerEffects` | 0 |
| `server:modules/match/engine/phase/gameover/phase.go:98` | `NewGameOverPhaseState` | 1 |
| `server:modules/match/engine/phase/lobby/debug.go:13` | `LobbyPickingPhaseState.DebugForcePickSpellsForAllPlayers` | 0 |
| `server:modules/match/engine/state/player_state.go:113` | `PlayerState.TakeDamage` | 2 |
| `server:modules/match/engine/state/player_state.go:160` | `PlayerState.TakeDamageWithSkillGain` | 0 |
| `server:modules/match/engine/state/player_state.go:170` | `PlayerState.TakeDamageWithSkillGainAndEvents` | 0 |
| `server:modules/match/engine/state/player_state.go:200` | `PlayerState.AddMana` | 1 |
| `server:modules/match/engine/state/player_state.go:216` | `PlayerState.ReduceMana` | 1 |
| `server:modules/match/engine/state/player_state.go:231` | `PlayerState.HaveMana` | 0 |
| `server:modules/match/engine/state/player_state.go:279` | `PlayerState.GetSpellSlots` | 0 |
| `server:modules/match/engine/state/player_state.go:283` | `PlayerState.HasSpell` | 0 |
| `server:modules/match/engine/state/player_state.go:322` | `PlayerState.AddEffect` | 6 |
| `server:modules/match/engine/state/player_state.go:388` | `PlayerState.SetCastingSpell` | 0 |
| `server:modules/match/engine/state/player_state.go:433` | `PlayerState.TriggerMageryGain` | 0 |
| `server:modules/match/engine/state/player_state.go:454` | `PlayerState.GetSkillGains` | 0 |
| `server:modules/match/engine/state/player_state/snapshot.go:17` | `PlayerSnapshot.IsEqual` | 0 |
| `server:modules/notifications/rpc.go:112` | `SendFriendRequestNotification` | 0 |
| `server:modules/notifications/rpc.go:131` | `SendFriendAcceptedNotification` | 0 |
| `server:modules/notifications/rpc.go:150` | `SendMatchInviteNotification` | 0 |
| `server:modules/notifications/rpc.go:170` | `SendMatchResultNotification` | 0 |
| `server:modules/notifications/rpc.go:203` | `SendSystemNotification` | 0 |
| `server:modules/notifications/rpc.go:224` | `SendFriendOnlineNotification` | 0 |
| `server:modules/notifications/types.go:136` | `CreateNotificationContent` | 0 |
| `server:modules/progression/magic_points.go:32` | `CalculateTotalMagicPoints` | 0 |
| `server:modules/progression/spell_slots.go:41` | `GetSpellSlotProgress` | 0 |
| `server:modules/progression/xp.go:30` | `CalculateLevelFromXP` | 0 |
| `server:modules/progression/xp.go:49` | `ProcessLevelUp` | 1 |
| `server:modules/race/db.go:100` | `FindById` | 0 |
| `server:modules/race/registry.go:80` | `Registry.Set` | 10 |
| `server:modules/skills/triggers.go:5` | `TriggerMeditationGain` | 0 |
| `server:modules/skills/types.go:39` | `NewSkillGains` | 0 |
| `server:modules/skills/types.go:60` | `SkillGains.Get` | 0 |
| `server:modules/spell_system/db.go:13` | `GetSpells` | 0 |
| `server:modules/spell_system/registry.go:39` | `SpellRegistry.GetByLevelRequirement` | 0 |
| `server:modules/spell_system/registry.go:61` | `SpellRegistry.Add` | 0 |
| `server:modules/spell_system/registry.go:65` | `SpellRegistry.LoadFromFile` | 0 |
| `server:modules/spellbook/validate.go:13` | `ValidateSpellExists` | 0 |
| `server:modules/spellbook/validate.go:26` | `ValidateSpellNotLearned` | 0 |

## Framework exclusions — keep

- Plugin `main.InitModule`, all registered module initializers/RPC handlers and matchmaker/auth hooks.
- `MatchInit`, `MatchJoinAttempt`, `MatchJoin`, `MatchLeave`, `MatchLoop`, `MatchTerminate`, `MatchSignal` on retained match types: required by vendored `runtime.Match` (`runtime.go:934`).
- `OnStart`, `OnTick`, `OnEnd` of registered effect handlers; these are invoked through `EffectHandler`.
- `Tick` / `HandleMessage` of reachable phases; `DoneState` is the specific unreachable exception discussed above.
- `quietLogger.WithField`, `WithFields`, `Fields`: required logger interface implementation for the simulator.
- Test/fuzz entry points and simulator `main` are roots, not deletion candidates.

## Source of truth in code

- `server:modules/main.go` and module `init.go` files — plugin roots, RPC/match registration.
- `server:vendor/github.com/heroiclabs/nakama-common/runtime/runtime.go` — callback interface contract.
- `server:modules/combat/**`, `modules/skills/**`, `modules/character/character.go` — legacy formulas and retained effective stats.
- `server:modules/match/engine/core/player_setup.go`, `state/player_state.go`, `state/actions.go`, `state/match_log.go`, `phase/game/phase.go` — live state/actions and unused plumbing.
- `server:modules/spell_system/spell_effects/**` — actual effect processing and lifecycle sink.
- `server:modules/notifications/**`, `modules/social/**`, `modules/endless_story/**` — helpers, registrations and scaffolding.
- `server:modules/character/rpc.go`, `modules/progression/**` — duplicated progression logic.
- Every exact declaration location is listed in Appendix A; supporting contracts: [[server-architecture]], [[spell-system]], [[progression]], [[notifications]], [[server-tutorial]].
