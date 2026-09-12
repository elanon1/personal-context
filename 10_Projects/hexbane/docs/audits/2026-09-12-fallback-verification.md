---
type: project
project: Hexbane
area: audits
status: verified-local
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, fallback, verification]
---
# Fallback local verification

> Deployment update: user explicitly requested production deployment and activation on2026-09-12, confirming that old-client compatibility is not required. Server commit `b2e7fe2862385d7bd5d756d73c2415804694c1ff` pushed to main; CI and deployment completed successfully. Both flags enabled; public fallback27183ms with reconnect/lost-result/actions PASS. Earlier local-only statements below are the pre-deployment verification record. Full deployment evidence: [[infra-and-deploy]].


Backend `feat/natural-fallback-player`, working tree implementation, no deployment/push. Server source baseline `f64b8fe`; client pre-existing dirty files preserved. Verification date2026-09-12. Runtime Nakama3.27.0 with Linux plugin built through `make build`; local Godot.NET SDK4.5.2/.NET9 compiled.

| Verification | Result |
|---|---|
| `go test ./...` | PASS |
| `go vet ./...` | PASS |
| `QUEUE_TEST_DB_URL=<disposable> go test -race ./modules/matchmaking/...` | PASS; real PostgreSQL concurrent arbitration/admission/expiry/cleanup |
| `TEST_DB_URL=<persona_review> go test -tags=integration -race ./modules/bot_persona/...` | PASS; persona leases, repeats, concurrent acquisition, level and nickname collisions |
| `MATCH_RESULT_TEST_DB_URL=<disposable> go test -race ./modules/character ./modules/match/engine/phase/gameover -count=1` | PASS; concurrent reward once, immutable result, foreign access denial, rollback |
| `TEST_DB_URL=<persona_review> go test -tags=integration -race ./modules/match_actions/...` | PASS; private temporary-table fixtures |
| `go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim` | PASS |
| `make build` | PASS, Linux ELF plugin |
| `dotnet run --project Tests/Matchmaking/Matchmaking.csproj` | PASS; cancel/join cleanup, late poll, accept, generation changes, replaced socket |
| `dotnet build hexbane.csproj --no-restore -v:q` | PASS, 0 errors /9 existing warnings |
| Server/client changed-path `git diff --check` | PASS |
| `golangci-lint` | Not installed; not claimed as run. Go vet ran. |

Isolated containers used only: `hexbane-queue-test` (Postgres127.0.0.1:55433) and `hexbane-fallback-live` (Nakama127.0.0.1:57350). Separate databases: queue per-test schemas, persona_review, match_actions_test, fallback_live. Application migrations1–8 applied on fresh isolated Nakama schema. Production and original Compose services untouched. Persona test initially pointed at an empty public schema and correctly failed for missing migration7; rerun against prepared separate persona_review passed. No production data repair was involved.

## Complete local runtime flows

`scripts/test_combat_runtime.mjs` now drafts only opcode70-owned spells, handles lobby/HUD readiness and supports fallback/custom-pvp.

- Fallback: `TEST_RACES=dark_elf TEST_DROP_RESULT=1 ... fallback`: queue wait25342ms, victory,112 snapshots/247 events, replacement socket rejoin succeeded. Test withheld opcode50 from result handling and fetched identical saved payload through get_match_result.
- Custom PvP: `TEST_DROP_RESULT=1 TEST_MATCH_ACTIONS=1 ... custom-pvp`: waits17ms/28ms, defeat/victory,72/71 snapshots, reconnect succeeded and first player’s deliberately lost result recovered. Duplicate add_friend/report actions returned stable receipts.
- Separate fallback post-match check: persona add_friend/report stable on retries; stranger could neither read the result nor act on that match; wire had no opponent_kind.
- Earlier fallback smoke also passed at20367ms before final result/action additions. This is historical supporting evidence, not the final binary check.

Meaningful regression failures observed before fixes: bot first-tick pick, unowned draft selection, immediate ready, plain struct parameters rejected by live MatchCreate, human PvP terminated by nonexistent persona heartbeat, cancellation returning before pending Join cleanup. Review additionally fixed stale owner blocking ranked, stale offer countdown cancelling a requeued search, transport replacement at join and stale result-recovery continuations.

## Behavior calibration

Machine-readable report: [[2026-09-12-fallback-ai-calibration.json]]. 1000 simulated fights per tier (3000), six races, all14 spells used per corpus; separate1000 seed samples. Timeout45.6% novice /49.2% regular /51.2% experienced. Full-match cast signatures1000/1000 distinct; these do not measure ordered opening diversity. Strong healing and symmetric loadouts are a plausible explanation for timeouts, not a proven diagnosis. The fixed human-vs-elf diversity fixture had a strong seat/race/build bias (seat1 won718/718 decided fights).

Existing combat tick benchmark on Apple M3 Pro: ~4.4–5.3us/op serial/parallel active scenarios; 100 resident matches benchmark is CPU simulation, not100 concurrent Nakama network duels. Neither measures scorer p99 or production capacity.

## Remaining release gates

- Human playtests, difficulty/tempo calibration and race/deck bias diagnosis. No claim of indistinguishable play.
- Rendered desktop/physical Android smoke for cancel/requeue, accept retry, result recovery and post-match actions. Compile/protocol checks do not prove UI/device behavior.
- Full distributed failure/soak test (node/process death, DB outage, exact deadline races through live transport,100 simultaneous network matches) and isolated scorer p99.
- Ordered opening-sequence metrics. The provided diversity proxy is explicitly weaker.
- Shared persona/character nickname uniqueness at concurrent creation, bounded history retention policy, and unified pending friend/profile product surface.
- Production deployment/activation was completed in the later authorized step (see below). Installed client distribution remains separate.

## Subsequent authorized production verification

CI run34699124648 PASS; server mainb2e7fe2, GitOps99aebe5, imagesha-b2e7fe2. Argo Synced/Healthy/Succeeded; pod1/1ready,0restarts. Migration8 clean,180personas. Public fallback27183ms, defeat,124snapshots/218events, reconnect and lost-result recovery PASS, repeated opponent actions PASS. Technical account removed,9users/1character retained. Full backup/deployment/rollback evidence: [[infra-and-deploy]]. Defaults stay false in the source example; production Helm explicitly overrides both flags to true at the user’s request.

## Source of truth in code
- server: `modules/matchmaking`, `modules/bot_persona`, `modules/match_actions`, `modules/character/{rewards,match_result}.go`
- server: `modules/match`, `cmd/duel-sim`, `scripts/test_combat_runtime.mjs`, `db/migrations/000006*..000008*`
- client: `Application/ArcaneDuel/Normal`, `Game/ScenesV3/{Dashboard/ModeOverlay,GameOver/GameOverScreen}.cs`, `Tests/Matchmaking`
