---
type: project
project: Hexbane
area: plans
status: implementing
created: 2026-09-08
updated: 2026-09-12
verified: 2026-09-08
tags: [hexbane, matchmaking, ai, implementation-plan]
sources: ["server:modules/match", "server:data/spells", "client:Application/ArcaneDuel/Normal/MatchManager.cs"]
---
# Natural fallback opponent — implementation plan

> Implementation update 2026-09-12: core queue/persona/lobby/AI/client flow is implemented locally. User confirmed **15–30 s**. The checklist below is the original design/verification plan, not a claim every rollout gate passed. Current executable contract: [[fallback-opponents]]; actual evidence and deviations: [[2026-09-12-fallback-implementation-progress]]. Existing duel_v2.4 progression and `character_match_rewards` supersede the older baseline and proposed duplicate ledger.


> **For agentic workers:** Use `superpowers:executing-plans` to implement task-by-task after design review. Checkboxes track execution; all are intentionally unchecked. This plan does not authorize deployment.

**Goal:** Provide a natural, legally built fallback opponent after a server-measured wait in the normal queue.

**Architecture:** PostgreSQL arbitrates one assignment per user; Nakama runs the shared authoritative match. Pure utility AI consumes delayed projected observations and submits normal combat commands. Stable personas own identity and build; match-local memory owns decisions and timing.

**Tech Stack:** Go 1.24.3, existing Nakama runtime/pluginbuilder, PostgreSQL, Godot C# client and existing test/simulation tools. No external inference service or new runtime dependency required.

**Spec:** [[2026-09-08-fallback-player-design]] (read the complete specification first).

## Global constraints

- Scope: normal unranked queue; ranked fallback disabled; current human XP/daily/skill rules preserved.
- Timing: 100 ms authoritative tick, 200 ms snapshots, 35 s aggregate draft timer per player, 180 s combat; no sleeps inside callbacks.
- Fallback delay: triangular 15/22.5/30 s; searching/allocation lease 10 s; offer TTL 20 s; default feature flag false.
- Go 1.24.3; format with gofmt; Linux pluginbuilder and Nakama versions stay aligned.
- Do not overwrite existing uncommitted combat restoration or client VFX work. Record starting git status; make focused commits only for newly implemented changes.
- Read/write docs only in the Hexbane vault. Update `updated`/`verified`, index and session journal with each implementation slice; describe proposed contracts as proposed until implemented.
- No hidden-state access, extra stats, altered RNG outcomes, synthetic chat or fabricated presence. Do not expose seeds/persona implementation fields to clients.
- This is an ordered engineering plan. Interface sketches below define boundaries; they are not a ready-to-apply implementation patch. Live integration and calibration remain required.

## File and dependency map

New server packages:
- `modules/matchmaking/{init,rpc,types,db,service,config}.go`: queue RPCs, assignment transitions and configuration. No combat decisions.
- `modules/bot_persona/{types,db,names,build,config}.go`: stable identity, leases and legal builds.
- `modules/match/engine/bot_ai/{types,perception,brain,utility,intent,timing,mistakes,draft}.go`: pure bot logic; no Nakama/DB/state imports.
- `modules/match/engine/phase/game/bot_adapter.go`: projection from engine to pure AI and command submission.
- `modules/match/engine/state/bot_state.go`: per-match bot memory reference and runtime deadlines, excluded from JSON.
- `modules/matchmaking/result.go`: idempotent result ledger helpers and participant result lookup.

Migrations: paired `db/migrations/000004_fallback_matchmaking.{up,down}.sql` for queue/assignments/personas and `000005_match_results.{up,down}.sql` for result ledger. These are next available numbers at planning time; recheck before creation and choose the next unused pair if another task adds migrations.

Configuration/data: `.env.dist` non-secret feature defaults; `data/bots/{names,personas}.yaml` validated curated content. This is separate from the strict flat spell catalog loader.

Existing integration: `modules/main.go`; `modules/match/normal_match/{init,matchmaker,join,signal}.go`; `modules/match/ai_match/{bot,init,join}.go`; `modules/match/engine/core/{loop,rejoin}.go`; `modules/match/engine/phase/{connecting,lobby,game,gameover}/phase.go`; `cmd/duel-sim/{main,fixtures}.go`; `scripts/test_combat_runtime.mjs`.

Client: modify `Application/ArcaneDuel/Normal/MatchManager.cs`, add `Application/ArcaneDuel/Normal/QueueClient.cs`, `Core/Match/QueueAssignment.cs`, modify `Game/Autoloads/MatchContext.cs` only where needed for ordinary fallback routing. Audit `Game/ScenesV3/{Lobby,GameOver}` and social/profile call sites before changing them. Explicit BotMatchManager remains an explicit training/test entry point.

Dependencies: 1 → 2 → 3; 4 → 5 → 6; 3 + 6 → 7 → 8. Each task ends with a focused reviewable commit, not a deployment.

## Task 1 — Atomic queue state machine and persistence

**Files:** new matchmaking types/db/service/config + migration 000004 queue/assignment tables; tests beside source (`service_test.go`, `db_integration_test.go`). Read [[matchmaking]], [[database]], [[rpcs]].

**Interfaces:** `Service.Join(ctx, userID, sessionID, JoinRequest) (QueueView, error)`, `Status(ctx, userID, QueueRef) (QueueView, error)`, `Cancel(ctx, userID, QueueRef) (QueueView, error)`, `Accept(ctx, userID, assignmentID) (QueueView, error)`, `Decline(ctx, userID, assignmentID) (QueueView, error)`; all on `*Service`. Define `QueueRef{QueueID string; Generation int64}`, `JoinRequest{RequestID, Mode string; Protocol int}`, `QueueView{QueueID string; Generation int64; State string; AssignmentID, MatchID string; ExpiresAt time.Time; PollAfterMS int}` with spec JSON tags and omission of absent offer fields.

- [ ] Write table-driven state tests before implementation. Cases: repeated join request returns same generation; second concurrent request does not open another active ticket; no fallback at 14.9 s; eligibility at its sampled deadline; expired heartbeat prevents matching; human pair wins before fallback reservation; canceled generation never publishes; stranger cannot read/accept/decline. Inject time and sampling.
- [ ] Run `go test ./modules/matchmaking -run 'TestQueue|TestAssignment'` and confirm missing behaviour fails.
- [ ] Implement explicit transitions using pool-scoped transactional advisory locks and a partial unique active-user index. Use DB time, ordered row locks, named states, allocation generation and CAS publication. Pool is mode/protocol/region, not per-level, so widening does not cross locking domains.
- [ ] Add real PostgreSQL interleaving tests, not mock-only transaction assertions: 100 contenders for one user, cancel while reserved, two simultaneous eligible tickets, late creator after allocation expiry, status after dropped response. Verify one valid assignment and no resurrected generations.
- [ ] Run focused tests + migration up/down/up on a disposable local DB. Add proposed queue RPC section to vault rpcs and new `docs/server/fallback-opponents.md`; index it. Commit `feat(matchmaking): arbitrate normal queue assignments`.

Transaction skeleton (design pseudocode):

```text
BEGIN
  lock(pool)
  expire leases and invalid reservations in a bounded batch
  load owned ticket and compatible live tickets oldest first
  reserve a human pair if available, otherwise eligible fallback
  set assignment + generation + allocation lease
COMMIT
create match outside transaction
BEGIN
  lock(pool)
  publish only where assignment, generation, lease and state still match
COMMIT
```

## Task 2 — Allocation, RPCs and admission

**Files:** matchmaking init/rpc; normal_match init/join/signal/matchmaker; main.go; focused `allocation_test.go`, `join_test.go`, `rpc_test.go`. Consumes Task 1 service; produces authoritative published match IDs.

- [ ] Test with a fake creator that blocks: cancel, expire, create late, fail creation, lose RPC response. Assert no obsolete ID can join and retry returns the published offer.
- [ ] Run `go test ./modules/matchmaking ./modules/match/normal_match` to confirm new assertions fail before implementation.
- [ ] Register the five RPCs from the spec. Authenticated context owns user/session; validate bounded JSON, mode, protocol and UUIDs; apply per-user request limits. `queue_status` advances eligible work and returns durable assignment state.
- [ ] Define `MatchCreator` as `Create(ctx context.Context, assignmentID string, generation int64) (string, error)` and use an adapter wrapping `nk.MatchCreate(ctx,"normal",params)`. Params contain assignment ID/generation and server-supplied allowed human IDs. Match state retains this admission context.
- [ ] Make first join require published exact match ID/generation + accepted assigned user; preserve already-joined reconnect. Protect both legacy decline signalling and new decline paths with participant/assignment ownership. Synthetic participant IDs never socket-join.
- [ ] Reject legacy `MatchmakerAdd` for enabled custom-queue cohort. Keep old clients explicitly isolated during transition. Match allocation caps and failure backoff leave tickets searching, while killed/unused allocations are reaped.
- [ ] Test accepted offers near the 20 s deadline against the 30 s empty-join grace, rejoin with replaced session, spoofed assignment and two human accepts. Commit `feat(matchmaking): create and admit assigned duels`; update [[matchmaking]], [[rpcs]], [[server-architecture]].

## Task 3 — Stable personas, legal builds and attribution

**Files:** bot_persona package, persona portion of migration 000004, data/bots files, normal_match init/join, ai_match bot builder, state/bot_state.go; tests `names_test.go`, `build_test.go`, `lease_integration_test.go`.

**Interfaces:** `PersonaStore.Acquire(ctx, assignmentID, userID string, level int) (Persona, error)` and `Release(ctx, assignmentID string) error`. `Persona` contains ID/name/race, level, base stats, three skills, learned spell IDs, style seed and version. `BuildPlayer(persona Persona) (*state.PlayerState, error)` lives in the match integration adapter (avoid making pure persona generation depend on the engine). Lease extension happens through live assignment/match lifecycle; completion and allocation expiry release leases idempotently.

- [ ] Add fixtures for six legal level-1 races and level bands 4/8/12/30; test effective race minima/maxima, earned points/MP, skill caps, owned starters and race slot bonus. Test that human and bot with identical legal stats derive identical combat profile values.
- [ ] Test name grammar weights, rune length 3–20, normalized uniqueness, reserved names, collision retries/spare exhaustion, stable repeated lookup and anti-repeat acquisition history. Use a seeded ≥10,000-name draw with tolerances rather than exact histogram equality.
- [ ] Run `go test ./modules/bot_persona ./modules/match/ai_match` and confirm regression tests fail before wiring generation.
- [ ] Generate curated versioned personas; use normal UUIDs and immutable public identity. Validate catalog at load. At assignment choose a legal nearby level/build; initialize standards only in selected loadout and carry legal ownership separately. Remove fallback's hard-coded `0000`, `Bot` and human-slot mirroring.
- [ ] Keep `IsBot` and `opponent_kind` server-side, with profile resolution routed through a match-authorized application lookup. Verify public DTOs do not serialize persona seed, internal mode or DB metadata.
- [ ] Test persona lease race and recovery from match termination/failed creation. Update [[progression]], [[combat-stat-rules]], fallback note; commit `feat(bots): add persistent legal opponent personas`.

## Task 4 — Shared legal draft and delayed readiness

**Files:** lobby phase/utils and new `bot_draft_test.go`; connecting phase; state/bot_state.go; new pure bot_ai draft/timing/types; ai_match init/join.

**Interfaces:** pure `DraftDelay(remainingTicks int64, picksLeft int, first bool, sampleTicks int64) int64`. Turn ordinal and deadline belong to match-local state, captured on transition to a bot turn. `SelectSpell` remains the authoritative mutation/validation entry point for both controller types.

- [ ] Add boundary tests using the actual 10 Hz clock:

```go
func TestDraftDelayReservesFuturePicks(t *testing.T) {
    // 35 seconds, seven picks, six reserves of .8 s plus 1 s margin.
    if got := DraftDelay(350, 7, true, 400); got != 292 {
        t.Fatalf("got %d, want 292 ticks", got)
    }
}
func TestDraftDelayExhaustedBudget(t *testing.T) {
    if got := DraftDelay(5, 1, false, 30); got != 0 {
        t.Fatalf("got %d, want immediate legal auto-fill", got)
    }
}
```

- [ ] Add phase tests: no first-tick pick with healthy budget, exactly one pick at deadline, consecutive same-player turns schedule distinct deadlines, reconnect preserves deadline, seven-slot full draft stays within aggregate timer, opponent picks influence only subsequent choices.
- [ ] Run `go test ./modules/match/engine/phase/lobby ./modules/match/engine/phase/connecting ./modules/match/engine/bot_ai` and confirm missing behaviour fails.
- [ ] Unify selection checks, including ownership; apply sampled utility draft preference over legal spells; finalize selected loadout from recorded picks only. Schedule acceptance/readiness and tick-based picks; keep shared loading/countdown unchanged.
- [ ] Re-run focused tests and existing lobby combat-restoration tests. Update [[op_03_lobby_update]], [[op_04_lobby_spell_selected]], [[op_05_lobby_spell_selected_update]], [[server-architecture]] with behaviour changes but no invented opcode changes. Commit `feat(bots): pace legal lobby drafting`.

## Task 5 — Enforced perception and reproducible command scheduling

**Files:** bot_ai types/perception/timing/brain; game/bot_adapter.go; game/ai.go and phase.go; tests `perception_test.go`, `bot_adapter_test.go`, `timing_test.go`.

**Interface contract:** `Observation` is a value snapshot with tick, own state, enemy public state and visible events; its structs/slices are deep-copied, with no `PlayerState` pointers. `Decision{Kind, SpellID string; DueTick int64}`. `Brain.Step(observation Observation) (Decision, bool)` advances match-local intent/timing; false means no command. `NewBrain(seed int64, profile Profile) *Brain`; profile holds the spec's numeric tier/style parameters. The adapter owns conversion to existing `CombatCommand` and `client_seq`.

- [ ] Write a dependency test rejecting imports from engine/state, phase/game, runtime or SQL inside pure bot_ai. Add paired runs with identical projected observations but different enemy hidden queue/book and combat RNG; decisions must agree.
- [ ] Test observed-at + minimum delay (rounded up), status/HP/mana perception lag, stale event expiry, planned action during recovery, rejected scheduled action, paralysis clearing queue, sequence monotonicity and reconnect preserving memory.
- [ ] Run `go test ./modules/match/engine/bot_ai ./modules/match/engine/phase/game` and confirm failures before integration.
- [ ] Replace global every-fourth-tick policy dispatch with due-tick polling and per-bot independent random streams. Preserve simulator legacy policies as explicit baseline options. Project the fields from recipient-visible protocol data; delay enemy observations and never hold mutable state aliases.
- [ ] Route every chosen cast/meditate/clear command through `Submit`; enforce execution validation unchanged. Collect reason-coded rejections and measured decision duration internally. No direct damage/effect mutation.
- [ ] Run focused and race tests; update [[combat-v2]], [[server-architecture]] and fallback note. Commit `feat(bots): enforce delayed public perception`.

## Task 6 — Utility, short plans and contextual mistakes

**Files:** bot_ai utility/intent/mistakes/brain; tests `utility_test.go`, `intent_test.go`, `mistakes_test.go`; game `bot_scenarios_test.go`; cmd/duel-sim main/fixtures.

- [ ] Write parameterized scenarios for all 14 spell rows in the design. Include: Dispel reflected by Mirror; Cleanse removes hex before poison; Consume Venom ignores enemy-owned poison; regeneration under poison; no reapply of active nonstacking status; paralysis immunity and too-late interrupt; heavy bolt versus a visible interruption window; heals before/after projected lethal impact; zero mana while poisoned.
- [ ] Add score ordering tests: useful heal outranks overheal at equal cost, lethal immediate damage outranks waiting for remaining poison ticks, useful reflection outranks duplicate reflection. These test decisions under conditions, not a copy of the scoring implementation.
- [ ] Run `go test ./modules/match/engine/bot_ai ./modules/match/engine/phase/game -run 'TestUtility|TestIntent|TestMistake|TestBot'` before and after implementation.
- [ ] Implement normalized weighted utility, top-three near-best sampling, 2–3-action plans, hysteresis and repetition memory. Read values through effective combat estimates and immutable catalog data. Candidate cap 16; no search over enemy private possibilities.
- [ ] Implement contextual error episodes with ≥5 s spacing, tier distributions and coherent personality bias. Keep tier fixed for the match. Track why a suboptimal option was chosen without exposing debug explanations to the player.
- [ ] Add simulator flags for AI version, tier, bot seed and persona; use current shared combat and race fixtures. Verify deterministic replay and independent combat RNG. Update fallback note; commit `feat(bots): add coherent imperfect combat decisions`.

## Task 7 — Uniform client flow, result recovery and social audit

**Files:** QueueClient/QueueAssignment + normal MatchManager; MatchContext where required; server matchmaking/result.go + migration 000005; gameover/phase.go; client tests under existing `Tests/`; extend `scripts/test_combat_runtime.mjs` for the new RPC flow.

**Interfaces:** C# `QueueClient.JoinAsync`, `PollAsync`, `CancelAsync`, `AcceptAsync`, `DeclineAsync` consume the exact Task 1 DTO fields via socket RPCs. Add `get_match_result {match_id}` returning the stored personalized opcode-50-compatible result only to an assigned human; not an unrestricted user/match query.

- [ ] Test client response reordering, double-click accept, cancel followed by late offer, socket replacement/account switch, reconnect, expired offer and leave. A single state owner drives Searching/MatchFound/Connecting regardless of opponent type.
- [ ] Replace new normal queue's AddMatchmakerAsync/ReceivedMatchmakerMatched route with QueueClient and cancellable serialized polling; preserve explicit training manager. Main-thread UI callbacks verify queue generation and socket identity.
- [ ] Add result-ledger tests: duplicate finalization gives one XP/skill/daily bonus mutation, rollback on partial failure, reconnect after opcode 50 loss retrieves the stored result, bots have no reward rows. Lock character row within the result transaction to serialize daily-bonus decisions.
- [ ] Implement ledger `(match_id,user_id)` primary key, immutable personalized result JSON and character update in one transaction. In-memory match ending retries persistence on transient error rather than granting a second result. Status/lookup retries return the committed payload.
- [ ] Audit all UI/profile/social paths for `IsAiMatch`, `Bot`, `0000`, presence count assumptions, opponent name resolution, separate scenes, result labels and friend/chat/report buttons. Fix application profile lookup before enabling affected screens; do not mask missing accounts with fabricated statistics. Record any unsupported surface as a rollout blocker.
- [ ] Validate PvP and fallback complete flows on local Nakama, cancellation, acceptance timeout and reconnect during each phase. Run `dotnet build hexbane.csproj --no-restore` in client and the project's corresponding automated tests. Update [[client-architecture]], [[matchmaking]], [[rpcs]], [[op_50_game_over]], [[shared-types]]; commit `feat(matchmaking): unify fallback client and result flow`.

## Task 8 — Verification, calibration and local rollout

**Files:** cmd/duel-sim and scripts/test_combat_runtime.mjs; tests added above; `.env.dist`; vault fallback note and this plan. No production deploy.

- [ ] Run `make test`, `go test -race ./modules/match/... ./modules/matchmaking/... ./modules/bot_persona/... ./cmd/duel-sim`, `make lint` (requires installed golangci-lint), then `make build`. Report actual output and any unavailable prerequisites; do not claim a missing check passed.
- [ ] Run ≥1,000 simulations per tier over race/deck fixtures and ≥1,000 seeds for timing/opening diversity. Save summarized machine-readable results under vault `docs/audits/` with AI/catalog/config version and seed list; no research documents in server repo.
- [ ] Profile 100 simultaneous local bot duels; record hardware, decision p50/p95/p99 and loop latency. Gate initially on decision p99 <1 ms and loop processing p99 <100 ms. Measure queue polling DB overhead as well; increase capacity only with evidence.
- [ ] Run local smoke cases: human before fallback deadline, human exactly at reservation, two eligible isolated humans, cancel/create interleaving, persona exhaustion, reconnect, DB error, killed creator and lost result packet. Verify orphan reaping and unchanged human-human match behaviour.
- [ ] Review replays and run human playtests described in the spec. Tune versioned parameters from observed mistakes/repetition/response timing; no claims of human indistinguishability from bot-vs-bot simulation.
- [ ] Enable only local test cohort with low concurrent cap, then expand only after metrics pass. Test kill switch: stops new fallback while existing matches finish and human matching remains available. Keep ranking disabled.
- [ ] Update all changed contracts, `verified`, `_index`, `_state` Decisions log and dziennik with actual results and unresolved release gates. Commit `test(bots): verify fallback lifecycle and behaviour`.

## Review checklist and handoff

- [ ] Product review of 15–30 s fallback, normal-only scope, human reward parity and profile/social policy.
- [ ] Queue concurrency/admission/result idempotency review.
- [ ] Legal persona generation and public-observation boundary review.
- [ ] Client presentation/reconnect and live spell scenario review.
- [ ] Calibration report accepted before rollout beyond local testing.

The plan is ready for implementation review. It does not claim executed tests, migrations, changed runtime behaviour or achieved concealment. Code work starts in a subsequent implementation task.

## Source of truth in code

Existing sources and inspected baseline are listed in [[2026-09-08-fallback-player-design]]. All new paths above are planned, not present at the time of writing. Server root: `/Users/elanon/GolandProjects/hexbane-server`; client root: `/Users/elanon/RiderProjects/hexbane`.
