---
type: project
project: Hexbane
area: plans
status: complete
created: 2026-09-15
updated: 2026-09-15
tags: [hexbane, commerce, cosmetics, HEX-23]
---
# Monetization foundation implementation plan

> For agentic workers: execute inline using executing-plans. User has authorized implementation of the Linear backlog; no commit, push or deployment is authorized.

**Goal:** Account-owned cosmetics can be equipped and rendered; a future verified purchase can grant an entitlement through a server-only idempotent API.

**Architecture:** A bounded server catalog defines stable item IDs and presentation keys. Account entitlements and loadout slots are stored transactionally. Public/private player views carry presentation only; combat never consumes cosmetics. XP acceleration applies only to remaining character-level progress, preserving normal post-cap study XP and skill gains.

**Tech stack:** Existing Go/Nakama/Postgres, Godot C# and source-generated System.Text.Json. No new service or payment SDK.

**Spec:** Linear HEX-23; existing contracts in server/progression, database, protocol/rpcs and client/design-system.

## Global constraints

- F2P with cosmetics (spell effects, skins, frames, fonts) and XP acceleration; no purchased combat modifiers or higher progression cap.
- No payment processing, prices, live grants or production rollout in this preparation task.
- Server validates ownership and slot compatibility; client never grants ownership.
- Unknown presentation keys fall back to defaults. Existing accounts use defaults without backfill.
- Preserve unrelated dirty work, including news integration. Documentation stays in this vault.

## 1. Persistent catalog and entitlements

Files: server `modules/commerce/{types,catalog,db,rpc,init}.go`, paired migration11, `modules/main.go`, commerce tests.

- [x] Add kinds `skin`, `frame`, `font`, `spell_effect`, `xp_boost`, stable IDs, display labels and local presentation keys. Defaults are free; premium examples remain locked unless granted.
- [x] Expose authenticated `get_collection` and `equip_cosmetic`. Empty item removes the slot; wrong kind, expired/unowned/disabled item and unsupported request fail without changing loadout.
- [x] Add server-only `Grant` with unique source receipt, account/product binding and atomic ownership update. Duplicate receipt is idempotent; reusing it for another account/product fails.
- [x] Tests: reject unauthenticated RPC, unknown kind/item, unowned equip; real isolated Postgres verifies reload, slot replacement, expiration, receipt retries and transactional rollback.

## 2. Match and reward integration

Files: player state/view types, shared player_setup, character/rewards and existing integration fixtures.

- [x] Load only active owned presentation into match state, propagate to both public/private views and reconnect paths. Nil/default profiles preserve older clients.
- [x] Read active XP acceleration inside the existing settlement transaction after receipt lookup. Apply bonus only below level cap, avoid boosting post-cap study or skills. Receipt retries preserve the first payout.
- [x] Tests: identical combat stats/spells with/without cosmetics; capped/near-cap/uncapped XP examples and repeated settlement with entitlement expiry.

## 3. Collection UI and rendering

Files: Core cosmetics DTOs, Application collection service, ClientJsonContext, DI, new collection modal, Dashboard shop wiring and shared cosmetic presentation.

- [x] Load server catalog and ownership on opening Shop; show owned/locked/equipped state, apply and remove buttons, retryable error. Keep purchasing unavailable until receipt verification is integrated.
- [x] Whitelist presentation keys for skin, frame, font and spell effect; render account selection in dashboard/collection and player selection in arena. Defaults remain faithful to existing visuals.
- [x] Tests: source-generated JSON under reflection-disabled AOT, actual UI selection/reload with two accounts, unavailable-server state, GPU visual comparison and mobile 960x432 fit.

## 4. Verify and document

- [x] Full Go tests, client build/AOT, Linux plugin build; isolated Nakama RPC and rendering acceptance, reviewer findings addressed.
- [x] Update commerce contract, rpcs, database, progression, relevant client notes, index and session log. Explain operator/verified-purchase grant boundary and no production rollout.

## Outcome

Implemented and reviewed. Runtime reconnect testing found missing identity replay; now10 precedes32. Same-ID cast regression found and fixed on the client. Verification evidence and the payment-provider boundary are recorded in [[commerce]]. The existing working copies and unrelated changes were preserved; no commits/push/deployment.
