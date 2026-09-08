---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, protocol, combat, duel_v2]
sources: ["server:docs/client/combat-v2.md", "server:docs/client/effect-sync.md", "client:docs/opcodes/duel-v2.md", "server:docs/match/communication.md"]
---

# Combat protocol 2 (`duel_v2`, catalog `duel_v2.4`)

Server-authoritative duel at **100 ms ticks** (`server:modules/match/engine/state/state.go:12`, `spell_system/version.go:7`). Opcodes 29–32 replace the retired 11–15/21–28. Client and server must ship together; the server tuple is `combat_protocol 2 / duel_v2 / duel_v2.4`.

## Handshake and loadout

1. `connecting`: server unicasts [[op_00_match_entry_data]] once per second with the version tuple, `tick_ms`, `me` (private) and `enemy` (public).
2. Client validates the tuple (`DuelVersion.Supported`, `client:Core/Match/DuelV2.cs:10`) and `tick_ms == 100`, then sends [[op_02_client_ready]] `{"combat_protocol":2,"user_id":…,"event_name":"match_entry_data"}`. A payload without `combat_protocol: 2` gets a private [[op_30_combat_result]] `{"kind":"protocol_rejected","required_protocol":2}` and the player never becomes ready (`server:modules/match/engine/phase/connecting/phase.go:119-125`).
3. When both are ready → [[op_01_server_ready]] → draft ([[op_03_lobby_update]], [[op_04_lobby_spell_selected]], [[op_05_lobby_spell_selected_update]], [[op_70_lobby_spellbook_spells]]) → [[op_06_launch_game]] → [[op_07_game_ready]] → [[op_10_game_data]] + [[op_16_game_countdown]] → `combat`.

Loadout rules (`server:modules/match/engine/core/player_setup.go:33-55`, `state/state.go:129-144`):

| Rule | Value |
|---|---|
| HP / mana | stat/race-derived maxima; combat formulas and growth in [[combat-stat-rules]] |
| `spell_slots` | min(learned non-standard spells, `max_spell_slots`) — what can actually be drafted |
| `max_spell_slots` | character entitlement: 3→6, Human 4→7, unlocks at levels 7/11/16; migrated slots may be grandfathered ([[progression]]) |
| standard spells | `magic_arrow`, `mirror_reflection`: always in `me.spells` and `me.standard_spells`, never draftable, no slot |
| bot (`ai_duel`) | copies the human's `spell_slots`/`max_spell_slots` and drafts random spells (`server:modules/match/ai_match/join.go:49-66`) |

Dedupe `spells` + `standard_spells` by id on the client (`DuelLoadout.DistinctById`).

## Catalog2.4 mechanics and progression boundary

Character resources and scaling use shared softened stats and bounded skill coefficients ([[combat-stat-rules]]). Two standards remain permanent. Arrow is fixed 1 damage and consumes mirrors; Mirror has a selected six-tier path determining its window, returned fraction and checkpoints. Entire hostile packages are intercepted, but selected tier<3 returns direct damage only. Original offense is retained through reflection, fraction and new-target mitigation apply once, and tiny reflected damage may round to0 (Arrow remains1). No reflection chain. `spell_reflected` can precede an impact with no returned damage when a status was blocked below tier 3. Clients must follow resulting events/snapshots rather than assume every reflection inflicts damage.

Reflection definition `value` is percentage; live `remaining` is one charge. Arrow break benefits fire before returned-target dodge. Conditional refunds and shared tempo modify authoritative mana/action deadlines, not client prediction. Primary config/graph APIs and free stat redistribution are outside-match RPCs ([[rpcs]]); opcode29 remains cast/meditate/clear_queue.

Normal/ranked queues are separated by server-owned `queue` property/query. Ranked requires level 30 only, with invited joins revalidated. Skills/MP/collection/selected primary path do not gate access; no MMR is implemented. Completed match settlement is atomic and idempotent by character/match receipt. Deploy migration 000004,000005 and compatible client/server together when rollout is performed; this implementation has not deployed them live.

## Commands (opcode 29)

```json
{"client_seq":1,"kind":"cast","spell_id":"magic_arrow"}
```

Kinds: `cast` (needs `spell_id`), `meditate`, `clear_queue` (no `spell_id`). Max 1024 bytes, unknown fields and trailing JSON rejected. `client_seq` is positive and strictly increasing per player per match; retrying the last accepted sequence is a silent no-op; an older one is rejected `stale_command`. Sender identity comes from the socket, never the payload. Full validation table in [[op_29_combat_command]].

Tick semantics (`Advance`, `server:modules/match/engine/phase/game/phase.go:138-304`):

- A submitted command sits in `pending` until the next tick, then becomes the single `queued` action (`action_queued` result). A newer legal command replaces it. `clear_queue` empties it (`queue_cleared`) without touching the active cast.
- The queued action executes when the player is neither casting nor in recovery. `cast` re-checks the spell and mana at that moment; mana is spent at cast start (`mana_spent`, then `cast_started` with `cast_end_tick` and `recovery_end_tick`). Starting a cast stops meditation.
- Casts release automatically at `cast_end_tick` (`cast_released`); the impact lands `travel_time` later (public `pending_impacts` in the snapshot, `spell_impact` event). There is no manual release, no cancel of the active cast, no cooldowns, no auto-repeat.
- Recovery: `recovery_end_tick = cast_end + recovery_time`. Paralysis (`paralyze` effect) is the only cast interrupt (`cast_interrupted`, reason `paralyzed`); mana is not refunded; recovery is re-applied from the interrupt time (`state/actions.go:87-98`). While paralyzed the queue and pending command are cleared every tick (`phase.go:255-259`), and 3 s of immunity follow (`immune_until_tick`).
- Meditation: needs alive, not paralyzed, not poisoned, not casting/recovering, mana < max. 0.8 s warm-up (`meditation_ready_tick`), then scaled additional mana on top of scaled passive regeneration (see [[combat-stat-rules]]) (`state/actions.go:39-85`). Stops on cast start, on poison/paralysis, or at full mana (`meditation_stopped`).
- Bots submit the same commands through the same `Submit` path every 4th tick (`phase/game/ai.go`).
- Death is committed after the full impact batch of a tick (`DeferDeath`), so simultaneous kills are a draw.

## Server messages

| Opcode | Delivery | Content |
|---|---|---|
| 30 [[op_30_combat_result]] | reliable, acting player only | `action_queued`, `queue_cleared`, `action_rejected`, and the connecting-phase `protocol_rejected` |
| 31 [[op_31_combat_event]] | reliable, all presences | public events (table in that page) |
| 32 [[op_32_combat_snapshot]] | reliable, each presence separately | full state every 2 ticks (200 ms), on the final tick, and on rejoin |
| 50 [[op_50_game_over]] | unreliable, per player | personalized result; sent on the first `combat_end` tick, then the match terminates |

Both 30 and 31 share one `event_seq` counter, so gaps in the public stream are normal. Ticks in `*_tick` fields are match-relative (`server_tick` units); 0 = no deadline. Effects' `started_at`/`remove_after` are logical-clock timestamps (1970-based) and must not be compared with the device clock.

### Event kinds (31)

`duel_v2.4` retains existing kinds: `spell_impact` may carry `reason=dodged`; passive HP recovery emits `heal` with `reason=passive_regeneration`. Deadlines round up to the first processing tick. Private spell metadata includes resolved primary paths and player cost/cast/recovery adjustments; standard descriptions reflect selected effects. Prepared client support accepts 2.4 and retains 2.2/2.3 for existing local tutorial/preview.

`cast_started`, `cast_released`, `cast_interrupted`, `spell_impact`, `spell_reflected`, `effect_applied`, `effect_removed`, `damage`, `heal`, `meditation_started`, `meditation_stopped`, `mana_spent`, `mana_regenerated`, `match_ended`. `spell_reflected`: `player_id` = reflector (new owner), `target_id` = original caster; the mirror is removed with reason `consumed` and the reflected spell then impacts with the swapped owner/target. `damage.amount` = HP actually lost, `absorbed` = shield consumed; `heal.amount` = HP restored, `overheal` = wasted. Effect kinds use engine names: `shield` (Barrier), `reflection` (Mirror), `paralyze` (Paralysis), `poison`, `regeneration`, `delayed_hex`.

### Rejection reasons (`action_rejected.reason`)

Submit time (`server:modules/match/engine/phase/game/reasons.go`): `not_participant`, `combat_ended`, `invalid_client_seq`, `stale_command`, `unknown_command`, `unknown_spell`, `insufficient_mana`, `unexpected_spell_id`, `paralyzed`, `meditation_unavailable`.
Execution time (`castReason`, `phase.go:381-395`; meditation `:281`): `meditation_unavailable`, `unknown_spell`, `insufficient_mana`, `busy`, `paralyzed`, `dead`, `invalid_spell`.

### Effect removal reasons (`effect_removed.reason`)

`expired` (duration ran out, also hex detonation), `depleted` (shield reduced to 0 by damage), `consumed` (mirror used; own poison eaten by `consume_venom`), `cleansed` (`cure` removed hex or poison), `dispelled` (`dispel` removed reflection/shield/regeneration), `match_ended` (queue flushed at end). The constants `broken` and `cured` in `server:modules/match/engine/state/match_log.go:29,32` are never emitted.

### Match end

`IsOver` when any player is dead or the 180 s clock hits 0 (`phase.go:205-243`). `match_ended.reason`: `defeated` (exactly one alive), `timeout` (both alive, clock 0), `draw` (both dead). Effects, shields, casts and meditation are cleared; the final snapshot carries `over:true`; then `combat_end` sends opcode 50 and terminates the match.

## Reconnect

- A new socket that re-joins the same match id is accepted for any existing participant, even when the match is "full" (`server:modules/match/normal_match/join.go:60-63`, `ai_match/join.go:80-82`).
- `RestorePresence` (`server:modules/match/engine/core/rejoin.go`) swaps the presence and, if the phase supports it (only `combat`), unicasts a fresh [[op_32_combat_snapshot]] containing the recipient's `queued` action and `last_client_seq`. No character reload, no restart.
- Messages and leave notifications from the old session are ignored once the presence was replaced (`core/loop.go:119`, `normal_match/leave.go:23-26`), so a late `close` of the old socket cannot clear the new one.
- Client: `MatchContext.MatchId` and `DuelState` survive the socket loss; on `ConnectionEstablished` the match manager rejoins (`client:Application/ArcaneDuel/Normal/MatchManager.cs:72-81`, same in `Bot/BotMatchManager.cs`), `DuelState.Apply` resumes `client_seq` above `last_client_seq` and never resets to 1 for the same match id. A failed rejoin raises `OnMatchRestoreFailed` and the HUD keeps a retry button until the next snapshot.
- Pre-combat phases: a leave cancels a `normal` match ([[op_09_match_canceled]]); in `combat` the match continues and the absent player may return.

## Server smoke test

`NAKAMA_URL=http://127.0.0.1:57350 node scripts/test_combat_runtime.mjs ai|pvp` from the server repo (`server:scripts/test_combat_runtime.mjs` exists; contents not verified here). Client-side smoke tests are in [[duel-v2-client]].

## Source of truth in code
- `server:modules/match/engine/phase/game/{phase,reasons,apply_spell_effect,ai}.go` — command handling, tick loop, events, snapshot, AI
- `server:modules/match/engine/state/{actions.go,player_state.go}` — cast/meditate/regen rules
- `server:modules/spell_system/spell_effects/{queue,engine,events}.go` and `effect_handlers/*` — effect lifecycle and removal reasons
- `server:modules/match/engine/core/{loop,rejoin,player_setup}.go` — dispatch, rejoin, loadout
- `client:Application/Match/DuelProtocol.cs`, `client:Core/Match/DuelV2.cs` — client send/receive and sequencing
