---
type: project
project: Hexbane
area: server
status: implemented-local
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, matchmaking, fallback, ai]
---
# Fallback opponents

> Deployment update: user explicitly requested production deployment and activation on2026-09-12, confirming that old-client compatibility is not required. Server commit `b2e7fe2862385d7bd5d756d73c2415804694c1ff` pushed to main; CI publication/deployment in progress. Earlier local-only statements below are the pre-deployment verification record. Final deployment evidence belongs in [[infra-and-deploy]].


Implemented on backend branch `feat/natural-fallback-player`, with coordinated changes in the local Godot client. Production has not been changed. Design: [[2026-09-08-fallback-player-design]]; implementation evidence and remaining gates: [[2026-09-12-fallback-implementation-progress]].

## Queue and admission

Normal queue uses authenticated `queue_config`, `queue_join`, `queue_status`, `queue_cancel`, `queue_accept`, `queue_decline`. Ranked retains Nakama matchmaking and the level-30 gate. Training retains explicit `ai_duel`.

Each search gets a server-time triangular fallback deadline: minimum 15 s, mode 22.5 s, maximum 30 s. Compatible humans take priority before a synthetic allocation is reserved. Status polling (normally 1 s), database latency and match creation add to the visible wait; 30 s is not an unconditional UI deadline under failures/exhaustion. Queue pool is server region + mode + protocol. Queue RPCs, not per-search goroutines, drive progress.

PostgreSQL serializes reservation per pool. Creation runs outside the transaction; allocation generation and match ID fence publication and first admission. Only accepted, assigned humans may first join; synthetic UUIDs cannot join over a socket. Existing human reconnects restore their presence without re-accepting an expired offer. Cancellation/decline/timeout clean up reservations; request IDs and generation checks reject stale cancellation and late async responses. A counterpart can be returned to search with a higher generation of the same queue ID.

Leases: searching 10 s, allocation 10 s, offered acceptance 20 s, active match 120 s refreshed every second. Unjoined offers expire; abandoned matches use the existing 30 s first-join/5 s empty-after-join reaper. Opportunistic maintenance prunes at most 100 terminal records per minute/service, retaining at least 7 days. No background task per queued player.

## Stable personas

Migration 7 persists 180 immutable personas: six races × levels 1–30. Each has a UUID, a 3–20 character nickname, legal build, owned spell collection, primary paths, stable style seed and version. Acquisition stays within ±2 levels; repeat avoidance prefers a different persona among the last 20 opponents/7 days. Exhaustion retries queue allocation instead of admitting an illegal build. A persona cannot hold two live assignment leases.

Name weights: 50% plain, 22% numeric, 12% country suffix, 8% separator+number, 6% case/prefix variant, 2% country+number. The curated base dictionary has 240 entries; collisions and reserved/confusable names are filtered. Country suffixes are only nickname text; they do not assert geolocation. Acquisition rechecks current character names. A concurrent character creation after this check can still choose that name: there is no shared cross-table uniqueness constraint.

`core.BuildBotPlayer` applies the current duel_v2.4 progression/combat rules, including race slots and owned optional spells. It starts with standards only; draft adds actual selected spells. Personas are not fake Nakama accounts. They earn no rewards, wins or invented account history. Internal flags/seeds are not included in public match DTOs.

## Lobby and combat

Connecting has a deterministic but variable ready delay. Current scheduling is relative to the match/human join tick; a separate persistent offer-publication acceptance timestamp from the original design is not implemented.

The first draft choice waits 2.5–6.5 s, later choices 1.2–4.0 s, with occasional additional pauses. Each choice has its own deadline even on consecutive turns. Near the phase deadline, remaining choices reserve time and a margin. Choices use owned spells, style preferences, duplicate-heal penalties and Poison/Consume Venom dependency. Bot and human picks pass the same ownership/slot/turn validation.

Combat uses pure `bot_ai` observations through a server adapter and submits ordinary combat commands. Enemy state is limited to what the game publicly reveals, delayed by the profile reaction window; own legal state stays current. No hidden enemy queue or server-state pointers enter the brain. The scorer understands all 14 spells, damage/heal value, shield, reflection, cleanse, delayed hex, paralysis, regeneration and venom consumption.

Profiles: novice reaction 0.5–1.1 s / mistake chance .15, regular 0.3–0.8 s / .08, experienced 0.3–0.6 s / .035. These are decision probabilities, not target match loss rates. Styles pressure/sustain/control are stable per persona; allocation seed changes match variation. Intents, short plans, weighted top-three selection, repetition penalties and contextual mistakes (at least 5 s apart) produce variation. Randomness uses separate seeded streams for reproducible diagnosis. Level ≤3 selects novice, ≥16 experienced, otherwise regular; this is initial configuration, not measured player skill.

Reflection penalizes hostile packages except the cheap Magic Arrow; visible self-heals do not trigger defensive reflection. Visible Delayed Hex is valued at detonation time, including Barrier defense. Planned queuing can occur before recovery ends, through normal command validation.

## Client and results

Both kinds of normal match use the same manager, lobby, countdown, duel and result scene. `QueueClient` fences late join/poll/cancel/accept replies by epoch, updates assignment generations, and retries a join on a replaced socket with cleanup on the old transport. Accept failure restores retry controls. Explicit training remains visibly separate.

Human rewards retain existing `character.SettleMatch` idempotency and current progression; there is no separate fallback reward formula. See [[op_50_game_over]] for result persistence/recovery and [[rpcs]] for post-match actions. Match actions are authorized by the completed receipt and use one response shape for both opponent types; pending persona requests are local records, not accepted friendships. The SDK friends list does not yet include those pending records. Automated correctness cannot establish that players will never recognize an automated opponent; behavior calibration and human playtests remain release gates.

## Local activation

Apply application migrations through version8 after Nakama migrations and deploy matching server/client versions. Set `HEXBANE_ENABLE_CUSTOM_QUEUE=true`, `HEXBANE_ENABLE_FALLBACK=true`, optional `HEXBANE_QUEUE_REGION=global`. Both flags default false in `.env.dist`; new normal clients first query `queue_config` and use built-in matching when custom queue is disabled. Old clients do not understand the custom queue; coordinate rollout. Flags are read at startup. An internal atomic fallback switch exists, but no admin RPC exposes it.

Turning fallback off keeps human custom matching available. Turning custom queue off returns new requests to built-in matching after server restart; drain active games before rollback. Do not roll back/drop migrations while matches or actions still reference them.

## Source of truth in code
- server: `modules/matchmaking`, `modules/bot_persona`, `modules/match/normal_match`, `modules/match/engine/bot_ai`
- server: `modules/match/engine/phase/{connecting,lobby,game,gameover}`, `modules/match/engine/core/bot_setup.go`
- server: `db/migrations/000006*`, `000007*`, `000008*`, `.env.dist`, `cmd/duel-sim`, `scripts/test_combat_runtime.mjs`
- client: `Application/ArcaneDuel/Normal/{MatchManager,QueueClient}.cs`, `Core/Match/QueueAssignment.cs`, `Tests/Matchmaking`, `Game/ScenesV3/Dashboard/ModeOverlay.cs`
