---
type: project
project: Hexbane
area: server
status: active
created: 2026-09-15
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, commerce, cosmetics, HEX-23]
---
# Commerce foundation

Migration11 seeds a server-owned SQL catalog, account entitlements, equipped cosmetic slots and immutable source-receipt bindings. This is preparation for verified payments; there are no prices, payment SDKs, purchase RPCs or client/admin grant RPCs.

## Collection RPC contract

Authenticated `get_collection {}` returns `{success:true,items:[{id,kind,title,presentation_key,owned,equipped,expires_at?}],loadout:{skin,frame,font,spell_effect},xp_boost_percent}`. The catalog response is limited to128 enabled entries, sorted by kind/default/id. `expires_at` is an optional RFC3339 UTC timestamp. Expired ownership is not active. Disabled entries are omitted; their saved loadout falls back to default.

`equip_cosmetic {kind,item_id}` returns the same full collection. Supported cosmetic kinds are `skin`, `frame`, `font`, `spell_effect`. Empty `item_id` or that kind's free default item removes the stored slot. XP boosts activate on grant and cannot be equipped. Authentication is mandatory (16); malformed/unknown/disabled/wrong-kind requests fail (3), unowned/expired equip fails (7), database unavailability fails (14). Payloads are objects, bounded to512 bytes, with unknown fields and trailing JSON rejected.

Loadout values are presentation keys, not product IDs. All four keys default to `default`. Seeded premium mappings: `skin_ashen→ashen`, `frame_arcane→arcane`, `font_cinzel→cinzel`, `spell_effect_arcane→arcane`. English titles are provided by the server; premium items start locked. Default IDs are `<kind>_default` and require no entitlement row.

## Server-only grant boundary

`commerce.Grant(ctx, db, userID, itemID, receiptID)` is an internal Go API for future trusted verification. Its caller must verify the payment provider and product/account binding first, and supply a globally namespaced provider transaction ID. Never forward a client assertion directly into this API.

An account lock serializes grants/equips. The unique receipt ID binds permanently to one account/product. An exact retry is a no-op even after expiry/disable; reuse for another account or product fails. Ownership and receipt commit together or roll back together. Receipt bindings remain after account deletion without blocking deletion; entitlements and equipped slots cascade. SQL foreign keys enforce loadout ownership and correct item kind. Transactions additionally validate catalog enabled state and active expiry.

Permanent cosmetics have no expiry by default; timed catalog entries use `duration_seconds`. A fresh grant of `xp_boost_25` provides25% character-level XP acceleration for24 hours. A new receipt extends remaining active time; retrying the same receipt never extends it. Multiple active boost products use the maximum percentage, not addition. Grants and catalog edits are server/operator responsibilities; no production grants were made.

## Match and progression

Shared `BuildPlayerState` reads the authoritative account collection after character acquisition. Both public and private `PlayerView` serialize additive `cosmetics`, including reconnect paths that reuse those views. Bots and older states may omit it; clients must fall back to defaults and whitelist presentation keys. No match request accepts cosmetic selections and no combat calculation consumes them. Selection changes take effect on the next match state construction.

After an existing settlement receipt lookup, the character settlement transaction reads active XP acceleration. For base XP `B`, current cumulative XP `X`, percent `P`, and cap6177:

`bonus = min(max(0,6177-X-B), floor(B*P/100))`

`reward.xp` is base plus bonus; optional `reward.xp_bonus` records just the bonus. The character cap remains6177. Normal study overflow and skill gains are unchanged. A120XP win at0XP grants150; at6030 grants147 (27bonus); at6100 or6177 grants120 with normal study overflow. Retried settlement returns the first stored receipt even after entitlement expiry.

## Validation and limits

Real isolated-schema PostgreSQL tests cover persistent equip/reload/removal, unknown/unowned/expired/disabled/wrong-kind rejection, receipt binding/retries, concurrent grants, injected-write rollback, SQL kind constraints, XP settlement at uncapped/near-cap/cap and receipt replay after expiry. View serialization verifies public/private cosmetics and unchanged existing fields/combat snapshots. This evidence does not establish payment-provider integration or production rollout.

## Source of truth in code

- server: `db/migrations/000011_commerce.{up,down}.sql`
- server: `modules/commerce/{types,db,rpc,init}.go`, `commerce_test.go`
- server: `modules/character/rewards.go`, `commerce_integration_test.go`
- server: `modules/match/engine/core/player_setup.go`
- server: `modules/match/engine/state/player_state.go`, `player_state/types.go`, `cosmetics_test.go`

## Integrated client and runtime acceptance (2026-09-15)

Dashboard Shop opens the new collection dialog. Typed `CollectionService` calls and all new DTOs use generated JSON metadata. Client keys are whitelisted: Ashen sprite tint, Arcane frame, Cinzel identity font and a thin cosmetic casting rune. Both local and enemy actors/HUD consume their own authoritative view. No resource path or ownership is accepted from client match input. Catalog entries can be added server-side using presentation keys shipped in the client; new assets/keys require a client release.

`test_collection_runtime.mjs` creates two disposable accounts only on localhost57350. An internal Go fixture calls the trusted Grant API against localhost55441, verifies identical receipts are no-ops and account transfer fails, and removes its temporary helper. Public equip requests cannot grant ownership. The test verified persistent equip, both initial peer views, actual game-data views and reconnect views. Reconnect now privately sends reliable opcode10 before32 from frozen match state. No payment-provider transaction was performed.

`MatchCosmeticsVerification.tscn` renders the actual captured opcode10 payload in the shared mobile arena: opponent identity, Ashen skin and rune are applied; screenshot also shows opponent frame and Cinzel name. Its same-action reconnect regression failed before resetting Player's cast identity on actor reload, then passed. `CollectionVerification.tscn` verifies touch equip, locked items, preview, fallback, interruption and retry at960x432. The displayed items are fixtures in that UI test; ownership/equip enforcement is separately checked through real RPCs.

Final validation: client build0 errors/11 existing warnings,179 reflection-disabled JSON/AOT checks, full Go suite including real isolated-schema PostgreSQL integration, Linux Nakama plugin build, two-account RPC/draft/combat/reconnect acceptance and GPU rendering. Reviewer findings addressed. Physical-device play and production deployment remain unperformed. Migration11 must precede deployment of the new server. Temporary test containers were removed after verification.
