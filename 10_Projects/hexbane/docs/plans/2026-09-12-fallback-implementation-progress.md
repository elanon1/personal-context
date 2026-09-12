---
type: project
project: Hexbane
area: plans
status: deployed-calibration-pending
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
---
# Fallback implementation ledger

> Deployment update: user explicitly requested production deployment and activation on2026-09-12, confirming that old-client compatibility is not required. Server commit `b2e7fe2862385d7bd5d756d73c2415804694c1ff` pushed to main; CI and deployment completed successfully. Both flags enabled; public fallback27183ms with reconnect/lost-result/actions PASS. Earlier local-only statements below are the pre-deployment verification record. Full deployment evidence: [[infra-and-deploy]].


Plan: [[2026-09-08-fallback-player-plan]]; spec: [[2026-09-08-fallback-player-design]].

User authorized implementation with 15–30 s fallback. Branch feat/natural-fallback-player in current clean backend checkout. Client has unrelated uncommitted work; preserve it.

Ruling: use 15/22.5/30 s triangular deadline — user changed interval, symmetric mode is a routine choice.
Ruling: preserve current progression2.4 and six redesigned races, ranked built-in queue and existing character.SettleMatch receipt ledger — these supersede the September8 baseline. Do not add obsolete XP/daily logic or duplicate result settlement migrations.
Ruling: implement new queue and AI in independent bounded agent tasks, root integrates lobby/client/lifecycle. Shared source paths explicitly assigned; no commits during concurrent writes.
Ruling: use existing backend checkout on a dedicated feature branch — backend baseline is clean; keep implementation visible in the user's workspace.

| Tasks | Shared contract | Resolution |
|---|---|---|
|1/2/3|allocation → persona → MatchCreate|Allocation ID/generation/humans/fallback/level/seed; root creator integrates|
|2/7|queue DTOs|five RPCs, authenticated identity, generation fencing; root client|
|3/4/5|bot runtime seed/style|MatchState.BotSeed/BotStyle; no wire fields|
|4/5|spell selection → observation|only actual drafted loadout; ownership validation|
|5/6|pure brain → game Submit|immutable public observation; independent RNG|
|7/existing rewards|settlement receipts|reuse current character.SettleMatch|
|1–8|test/migration paths|new migrations000006 queue,000007 personas; isolated SQL test DB|

## Implemented and verified locally

- Tasks1–2: queue arbitration, six RPCs including config, admission fencing, cleanup and migration6; real SQL race tests.
- Task3: persona persistence/leases/names/legal current builds, migration7, integration tests and independent review fixes.
- Task4: shared ownership validation, actual drafted loadout, tick-scheduled choices/readiness, time-budget tests.
- Tasks5–6: public delayed observation, reproducible brain/utility/14spells/context errors and ordinary Submit; unit/race tests.
- Task7: normal client queue flow, cancellation/requeue/socket fences, exact result saved atomically in existing receipt, authenticated get_match_result/recovery, post-match friend/report action abstraction and migration8.
- Task8: automated tests, Linux/plugin and client builds, full local fallback/PvP/reconnect/lost-result checks,3000+1000 simulation corpus. Production rollout completed after explicit authorization; UI/device playtests and calibration remain open.

Evidence: [[2026-09-12-fallback-verification]], [[2026-09-12-fallback-ai-calibration.json]]. Runtime contract: [[fallback-opponents]].

## Deliberate differences from the original plan

- Migrations6/7/8 were next free; reuse existing v4 reward receipt instead of a duplicate planned ledger. Stored opcode50 is an added JSON field, not a second table.
- Current progression/race/primary contracts supersede old baseline/daily-bonus tests.
- Connecting readiness is scheduled relative to join/match ticks, not a separately persisted offer-publication timestamp.
- Config flags are startup-read; the internal atomic fallback switch has no administration RPC. Invalid env config fails initialization rather than silently preserving a partial service.
- No match profile UI currently needs a new profile lookup. Result actions use authoritative stored opponent IDs. Persona friend requests remain local pending records and are not in the SDK list; no fake social activity or stats.
- Behavior report uses combat-valid simulator fixtures, not acquired SQL persona builds; complete persona legality is tested separately. Ordered opening diversity, scorer p99 and a100-network-match load test were not performed.
- Subsequent user instruction authorized server push/deployment and activation: server main `b2e7fe2`, GitOps `99aebe5`, both flags true, public smoke PASS. Client changes remain local and preserve pre-existing dirty work.

## Source of truth in code
- server: modules/match, modules/matchmaking, modules/bot_persona
- client: Application/ArcaneDuel/Normal
