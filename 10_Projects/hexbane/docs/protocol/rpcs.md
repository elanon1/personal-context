---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, protocol, rpc, nakama]
sources: ["server:RPCs.md", "server:docs/API-REFERENCE-v2.md", "server:docs/match/api-reference.md", "server:docs/progression/client/menu-rpc-requirements.md", "client:docs/Server/progression/menu-rpc-requirements.md", "server:docs/client/client-implementation-prompt.md"]
---

# RPC reference (verified against code, 2026-09-08)

Every RPC the Nakama plugin registers, with the exact JSON it accepts and returns. All handlers are
registered from `server:modules/main.go` via each module's `InitModule`. Unless stated otherwise:

- **Auth**: Nakama runs RPCs with a session token; every handler reads `RUNTIME_CTX_USER_ID`. The only
  handlers that do not touch the user id are `healthcheck`, `get_races`, `get_race`, `get_starter_spells`,
  `get_entry_spells`, `get_spell`, `get_spell_lore`, `get_spell_details_yaml`.
- **Errors**: handlers return HTTP 200 with `{"success": false, "message": "..."}` (or `"error"` for
  `learn_spell`). Matchmaking, match actions, result recovery and `tutorial` can return real gRPC errors. The gRPC error-code
  table in the old API-REFERENCE-v2 does not describe any other RPC.
- **Version stamp**: spell RPCs and `get_character_details`/`tutorial` embed
  `{"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.4"}`
  (`server:modules/spell_system/version.go:3-19`). The client supports server2.3 and local tutorial2.2; it rejects other values
  (`client:Core/Match/DuelV2.cs:8-12`) in `get_player_spells` and `tutorial` only.

## Client ↔ server coverage

| Client calls | Server registers | Status |
|---|---|---|
| `create_character`, `get_my_character`, `get_character_by_id`, `allocate_stat_points`, `get_character_details`, `get_progression`, `tutorial`, `set_tutorial_completed`, `get_starter_spells`, `get_player_spells`, `get_spellbook`, `get_spell`, `learn_spell`, `get_races`, `find_friend`, `remove_friend`, `decline_match`, `create_ai_arcane_duel` | yes | OK (see per-RPC notes; `create_character_match_story` is broken server-side) |
| `get_users` (`client:Application/Modules/Social/Queries/GetUsers/GetUsersQueryHandler.cs:44`) | **no** | dead client code, query never dispatched |
| `start_story` (client removed 2026-09-08) | **no** (commented out in `server:modules/endless_story/init.go:13`) | client removed 2026-09-08 |
| — | `debug_create_character`, `get_available_spells`, `get_entry_spells`, `get_my_spells`, `get_spell_lore`, `get_spell_details_yaml`, `get_race`, `create_playstyle`, `get_playstyles`, `update_playstyle`, `delete_playstyle`, `healthcheck`, `notifications_list`, `notifications_delete` | server-only (client uses Nakama SDK built-ins for friends/notifications) |

`RpcAsync` on the socket is used for `decline_match`, `create_ai_arcane_duel`; everything else goes through `client.RpcAsync(session, id, payload)`.

Prepared client primary/respec integration is described in [[duel-v2-client]]; deployment status is separate from this server contract.

## Character module (`server:modules/character/init.go`)

### `create_character`
- Registered `init.go:11`, handler `rpc.go:14`. Client: `client:Application/Modules/Character/Commands/CreateCharacter/CreateCharacterCommandHandler.cs:40-53`.
- Request (`types.go:4-12`):
  ```json
  {"name":"Merlin","avatar":"human","race_id":"human",
   "spell_ids":["firebolt","mend","poison","cleanse"],
   "base_strength":133,"base_intelligence":134,"base_dexterity":133}
  ```
- Validation order (`validate.go:83-103`): name 3–20 chars → `race_id` in registry → base stats sum to
  `progression.CreationPoints` = 400, each base stat ≥10; all races share these bounds and flat racial stat modifiers are zero → `spell_ids`: exactly
  `StartingSpellCount(race)` = 4 for `human`, 3 otherwise (`validate.go:106-111`), distinct, each
  `starter: true` and not `standard` (`validate.go:113-129`).
- One character per user: `FindByUserId` first; if one exists → `{"success":false,"message":"Character already exists","character":{...existing}}`.
- Character row and `character_spells` rows are written in one transaction (`db.go:49-75`); no MP charged.
- Response: `{"character": <Character>, "message":"Character created successfully", "success":true}`.
- Full contract in [[race-selection]].

### `debug_create_character`
- Registered only when `HEXBANE_ENABLE_DEBUG_RPCS=true` (`init.go:17-23`), handler `rpc.go:76`. Creates "Test" / `red_mage` / `race.DefaultRaceId` ("human") with 133/134/133 and the first N starters from the registry. Not in `docker-compose.yml`, so off by default.

### `get_my_character`
- `init.go:30`, handler `rpc.go:125`. Client: `client:Application/Modules/Character/Queries/GetCurrentCharacter/GetCurrentCharacterQueryHandler.cs:37` (RpcName default in `GetCurrentCharacterQuery.cs:12`).
- Request: ignored. Response: `{"character": <Character>|null, "message", "success"}`. No character → `"No character found for user"` (this string drives the client's create-character decision).

### `get_character_by_id`
- `init.go:25`, handler `rpc.go:157`. Client: `GetCharacterByIdQueryHandler.cs:38`.
- Request `{"character_id":"char_<user>_<unix>"}`. **No ownership check**: any authenticated user can read any character. Same envelope as `get_my_character`; not found → `"No character found for user"`.

### `<Character>` object (`types.go:23-67`, filled by `character.go:211-241`)
```json
{"id","user_id","name","avatar","race_id",
 "level","experience","experience_to_next",
 "base_strength","base_intelligence","base_dexterity",
 "strength","intelligence","dexterity",
 "skill_meditation","skill_spell_resistance","skill_magery",
 "spell_slots","magic_points","magic_points_spent","available_magic_points","unspent_stat_points",
 "wins","losses","tutorial_completed","created_at","updated_at"}
```
Client DTO `client:Application/Modules/Character/Dto/CharacterResponse.cs` expects `max_spell_slots` and
`experience_to_next_level`, which the server never sends (they stay 0); it ignores `spell_slots`,
`experience_to_next`, `available_magic_points`, `base_*`.

### `allocate_stat_points`
- `init.go:35`, handler `rpc.go:187`. Client: `AllocateStatPointsCommandHandler.cs:38-49`.
- Request `{"character_id":"(optional, ignored)","strength":2,"intelligence":3,"dexterity":0}`.
- Rules: each value ≥ 0; each positive value is applied in order STR, INT, DEX through
  `Character.AllocateStatPoints` (`character.go:140-177`): must not exceed `unspent_stat_points`
  remaining at that step. Race-specific floors/ceilings are removed; shared minimum 10 remains. The total
  does **not** have to equal all unspent points (that check is commented out, `rpc.go:231-235`).
  Row is locked `FOR UPDATE` in a transaction (`rpc.go:204-270`).
- Response `{"character": <Character>|null, "message", "success"}`. Errors: `"All stat values must be non-negative"`, `"not enough stat points"`, `"invalid point allocation"`, `"<stat> must be at least 10"`; active-match changes are rejected before mutation.

### `get_character_details`
- `init.go:40`, handler `details.go:156`. Client: `GetCharacterDetailsQueryHandler.cs:26-28`. Contract in [[character-details]].

### `get_progression`

Request ignored; authenticated current-character progression. Example level 1 non-Human:

```json
{"success":true,"message":"Progression retrieved successfully","level":1,"experience":0,
 "experience_to_next_level":45,"magic_points":0,"magic_points_spent":0,"available_magic_points":0,
 "spell_slots":3,"next_spell_slot_level":7,"unspent_stat_points":0,
 "study_xp":0,"ranked_eligible":false,"primary_tier":1}
```

Cumulative XP uses `T(L)=45*(L-1)+6*(L-1)*(L-2)`, cap 30/6177. `experience_to_next_level` never goes below 0 and is0 at cap. Next slot milestones are7/11/16, then0; Human gets one additional starting/capped slot and migrated earned slots are preserved. `study_xp` is post-cap remainder 0–499 (500→5MP). `primary_tier` is earned tier 1–6, independent of selected paths. `ranked_eligible` requires only level≥30. Skills, MP, collection and unallocated primary tiers never block it. See [[progression]] for rewards and migration. Source: `server:modules/character/rpc.go`, `progression_test.go`.

### `get_primary_progression` and `set_primary_path`

Authenticated, current character only. `get_primary_progression` accepts `{}`; `set_primary_path` accepts exactly `{"spell_id":"magic_arrow","path":["magic_arrow.v1.root","magic_arrow.v1.speed2"]}`. An empty path resolves to the free root. Unknown fields/trailing JSON, foreign/unknown nodes, skipped tiers, disconnected paths and paths exceeding earned tier are rejected (code 3). Both primaries are independent; no MP is spent.

Success envelope for both:

```json
{"success":true,"graph_version":1,"earned_tier":2,"primaries":[
 {"spell_id":"magic_arrow","path":["magic_arrow.v1.root","magic_arrow.v1.speed2"],
  "graph":{"version":1,"spell_id":"magic_arrow","nodes":[
   {"id":"magic_arrow.v1.root","tier":1,"next":["magic_arrow.v1.speed2","magic_arrow.v1.economy2"],"modifier":{}}]},
  "config":{"spell_id":"magic_arrow","version":1,"level":2,"cast_seconds":0.7,"mana_cost":5,"recovery_seconds":0.4,"fixed_damage":1}},
 {"spell_id":"mirror_reflection","path":["mirror_reflection.v1.root"],
  "graph":{"version":1,"spell_id":"mirror_reflection","nodes":[]},
  "config":{"spell_id":"mirror_reflection","version":1,"level":1,"cast_seconds":0.5,"mana_cost":9,"recovery_seconds":0.4,"window_seconds":1.5,"return_fraction":0.1}}
]}
```

Graph node arrays above are abbreviated for readability; production returns every node for both complete six-tier DAGs. Exact node suffixes, modifiers and optional config fields are in [[spell-system]]. `earned_tier` is derived from character levels1/5/10/16/23/30; `config.level` is selected prefix length. Config timing/cost is before profile/race adjustments; live spell metadata includes those adjustments.

### `respec_stats`

Authenticated current character only. Request `{"strength":133,"intelligence":134,"dexterity":133}` at level 1; replace values to sum exactly `400+5*(min(level,30)-1)` at later levels. Each stat must be at least10. No race ceilings or flat grants. Success `{"success":true,"character":<Character>}`; base and effective stats match, unspent points become0. No cost, skills/ownership/MP/path reset, or once-only restriction.

Both mutations (`set_primary_path`, `respec_stats`) lock the character row and reject a live match lease with Nakama code 9 / `finish the current match before changing your build`. Missing auth is 16, missing character 5, invalid payload/build 3. Client refreshes details after success and shows failures. Lease acquisition snapshots the current build for a match and has a 15-minute crash expiry. Source: `server:modules/character/primary_progression.go`.

### `tutorial` and `set_tutorial_completed`
- `init.go:45` / `init.go:49`; handlers `tutorial.go:106` / `rpc.go:153`. Contract in [[server-tutorial]].
- `set_tutorial_completed` always returns `{"character":null,"message":"Use tutorial progression; tutorial state cannot be reset","success":false}`. Client caller `SetTutorialCompletedCommandHandler.cs:31` is only reachable through `CharacterService.SetTutorialCompleted`, which nothing calls.

## Spellbook module (`server:modules/spellbook/init.go`)

### `get_starter_spells`
- `init.go:33`, handler `rpc.go:375`. Client: `client:Application/Modules/Spell/Queries/GetEntrySpells/GetEntrySpellsQueryHandler.cs:39`.
- Request ignored. Returns every catalog spell with `starter: true` (6 in `duel_v2.4`: barrier, cleanse, firebolt, heavy_bolt, mend, poison):
  ```json
  {"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.4","success":true,
   "message":"Starter spells retrieved successfully",
   "spells":[{"nature":"ember","incantation":["Tal","Rath"],"id":"firebolt","name":"Firebolt",
              "description":"...","school":"Fire","mana_cost":20,"cast_time":1.5,"icon_path":"firebolt"}]}
  ```
- `cast_time` is **seconds** (`types.go:67-76` copies `spell.CastingTime`). The client deserialises it
  into `Spell.CastingTimeMs` (`client:Core/Spells/Spell.cs:48-49`), so `CastDurationSeconds` is wrong by 1000× for starter spells.

### `get_player_spells`
- `init.go:27`, handler `rpc.go:292`. Client: `GetPlayerSpellsQueryHandler.cs:42` (checks version) and `GetSpellsQueryHandler.cs:42` (legacy, deserialises into `Core.Spells.Spell[]`).
- Request: ignored (client sends `character_id`/`category`; server uses the session user).
- Response:
  ```json
  {"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.4","success":true,"message":"...",
   "spells":[{"nature","incantation","standard":false,"starter":true,"recovery_time":1.0,"travel_time":0.4,
              "id","name","description","school","mana_cost":20,"cast_time":1.5,"icon_path",
              "is_learned":true,"magic_points_cost":5,"level_requirement":1}],
   "learned_spell_ids":["firebolt","mend","poison"],"spell_slots_used":3,"spell_slots_max":3}
  ```
- All times in **seconds**. `spell_slots_used` = learned count, `spell_slots_max` = `characters.spell_slots` (draft slots), so "used" can exceed "max": the collection is not capped by slots (`rpc.go:165-169`). Standard spells are listed with `is_learned` reflecting the DB (false unless learned).

### `get_spellbook`
- `init.go:9`, handler `rpc.go:14`. Client: `GetSpellbookQueryHandler.cs:37`.
- Response: `{"combat_protocol",…,"spells":[<full spell_system.Spell>],"count":N,"success":true,"message":"Spellbook retrieved successfully"}`. Full YAML shape: `id,name,type,school,icon,casting_time,recovery_time,travel_time,mana_cost,magic_point_cost,level_requirement,standard?,starter,effects[],description,flavor?,assets?,nature,incantation` (`server:modules/spell_system/spell.go:5-32`). Times in seconds. Learned spells only, no standards.
- Client DTO `client:Application/Modules/Spell/Dto/GetSpellbookResponse.cs` maps a singular `spell`, so the array is dropped on deserialisation.

### `learn_spell`
- `init.go:15`, handler `rpc.go:75`. Client: `LearnSpellCommandHandler.cs:37-44`.
- Request `{"spell_id":"heavy_bolt"}` (client also sends `character_id`, ignored).
- Checks in order: spell exists → not already learned → not standard → `level >= level_requirement` → `available_magic_points >= magic_point_cost` → insert + `magic_points_spent += cost` atomically (`db.go:248-293`).
- Response `{"success":true,"error":"","spell_id":"heavy_bolt","magic_points_remaining":0}`; on failure `"error"` is one of `Invalid request format`, `Authentication required`, `Database error`, `No character found`, `Spell not found`, `Spell already learned`, `Standard spells are already known`, `Level requirement not met`, `Not enough magic points`, `spell already learned`, `not enough magic points`. No slot check: learning is bounded by MP only.
- Every non-standard spell costs `magic_point_cost: 5` in the catalog; standards cost 0 (`server:data/spells/*.yaml`).

### `get_available_spells`
- `init.go:21`, handler `rpc.go:217`. No client caller. Non-standard catalog spells with `{spell_id,name,magic_point_cost,can_afford,already_learned}` under `spells`, plus version stamp.

## Spell system module (`server:modules/spell_system/init.go`)

| RPC | Registered | Handler | Request | Response |
|---|---|---|---|---|
| `get_spell` | `init.go:24` | `rpc.go:74` | `{"spell_id":"firebolt"}` | `{version…, "spell": <Spell>|null, "success", "message"}`; client `GetSpellQueryHandler.cs:38` |
| `get_entry_spells` | `init.go:12` | `rpc.go:14` | `{"school"?, "limit"?}` (ignored) | starters as `{version…, "spells":[<Spell>], "count", "success", "message"}`; no client caller |
| `get_my_spells` | `init.go:18` | `rpc.go:40` | `{"school"?, "limit"?}` | owned non-standard spells + both standards, sorted by id (`db.go:24-83`); no client caller |
| `get_spell_lore` | `init.go:9` | `identity.go:96` | ignored | `{"lore_version":1,"natures":[5],"words":[16]}`; no client caller |
| `get_spell_details_yaml` | `init.go:30` | `rpc.go:128` | `{"spell_id"}` | raw **YAML** text of the spell, or `error: "..."` YAML; intended for HTTP with server key; no client caller |

The catalog is loaded from `/nakama/spells` at startup (`init.go:36`), 14 spells in `server:data/spells/`.

## Race module (`server:modules/race/init.go`)

### `get_races`
- `init.go:21`, handler `rpc.go:12`. Client: `client:Application/Modules/Race/Queries/GetRaces/GetRacesQueryHandler.cs:37`.
- Request ignored. `{"races":[<Race>], "success":true, "message":"Races retrieved successfully"}`. `<Race>` shape in [[race-selection]]. Registry is loaded once at startup from the `races` table (`init.go:12-16`).

### `get_race`
- `init.go:26`, handler `rpc.go:34`. `{"race_id":"orc"}` → `{"race": <Race>|null, "success", "message"}`; errors `race_id is required`, `Race not found`. No client caller.

## Matchmaker queue (socket operation, not an RPC)

Use Nakama MatchmakerAdd string property `queue=normal|ranked`; omitted means normal. The server overwrites query to `+properties.queue:<queue>` and forces 2 players. Ranked validates level≥30 at ticket, matched entries and invited join; unknown/mixed queues fail. No rating/MMR/season system is introduced. Both queues use the same normal match module. See [[progression]].

## Match RPCs

### Normal queue RPCs (2026-09-12)

All require an authenticated Nakama user. Requests are strict JSON (unknown fields rejected); user/build/region/fallback status are not client input. Queue RPCs return real runtime errors for invalid request(3), denied admission(9), missing record(5), disabled queue(9), rate limit(8), unexpected server failure(13).

| RPC | Request | Response `data` |
|---|---|---|
| `queue_config` | `{}` | `{enabled:bool,poll_after_ms:1000}` |
| `queue_join` | `{request_id:UUID,mode:"normal",protocol:2}` | QueueView |
| `queue_status`, `queue_cancel` | `{queue_id:UUID,generation:int64}` | QueueView |
| `queue_accept`, `queue_decline` | `{assignment_id:UUID}` | QueueView |

Envelope: `{success:true,data:...}`. QueueView: `{queue_id,generation,state,assignment_id?,match_id?,expires_at?,poll_after_ms}`; expiry is an RFC3339 timestamp. States: searching/reserved/offered/accepted/joined/completed/canceled/declined/expired. Client polls at ~1s; stale Status may return a newer generation, stale Cancel cannot cancel it. Accept is idempotent; only accepted assigned users can socket-join. No `is_bot`/persona seed/opponent kind is returned. See [[fallback-opponents]].


### `create_ai_arcane_duel`
- `server:modules/match/ai_match/init.go:34`. Client: `client:Application/ArcaneDuel/Bot/BotMatchManager.cs:92` (socket RPC).
- Creates an `ai_duel` match; response `{"success":true,"message":"Match created","data":{"match_id":"<id>.nakama1"}}`; failure is a gRPC error from `MatchCreate`. See [[matchmaking]].

### `decline_match`
- `server:modules/match/normal_match/init.go:31-53`. Client: `client:Application/ArcaneDuel/Normal/MatchManager.cs:158-159`.
- `{"match_id":"..."}` → sends an authenticated `{kind:"decline",user_id}` signal; only invited/current participants before combat may decline; response `{"success":true,"message":"Match declined","data":null}`. gRPC errors `match_id is required`, `failed to decline match`. See [[matchmaking]].

### `create_character_match_story` (server prototype; client removed 2026-09-08)
- `server:modules/endless_story/create_character/init.go:30`. Client manager and keyed DI registration were removed on 2026-09-08.
- Calls `nk.MatchCreate(ctx, "create_character", …)` but the match handler is registered as `"v2_create_character"` (`init.go:25`), so the call fails at runtime. Story RPCs `start_story`, `create_character_story`, `continue_character_story` are commented out (`server:modules/endless_story/init.go:13-23`).

### `get_match_result`
- Authenticated request `{match_id:string}`; strict JSON, ≤4096 bytes, match ID 1–256 chars.
- `{success:true,data:<personalized opcode50>}` from the caller’s existing character receipt. No reward mutation on lookup. Missing, foreign or historical payload-less receipt → runtime NotFound(5); malformed→3, unauthenticated→16, internal→13.
- See [[op_50_game_over]]; server `modules/character/match_result.go`.

## Social module (`server:modules/social/init.go`)

### `find_friend`
- `init.go:12`, handler `rpc.go:11`, lookup `db.go:10-38`. Client: `client:Application/Modules/Social/Queries/FindFriend/FindFriendQueryHandler.cs:42-52`.
- `{"name":"Merlin"}` matches `characters.name` **or** `users.username` exactly. Response
  `{"success":true,"message":"Friend Found!","data":{"character_id","user_id","name","username"}}`.
  Errors (`"data":{}`): `Player not found`, `Cannot find yourself`, `Failed to find friend`, `Invalid request format`.

### `remove_friend`
- `init.go:17`, handler `rpc.go:308`. Client: `RemoveFriendCommandHandler.cs:36-42`.
- `{"user_ids":["…"],"usernames":["…"]}` (at least one) → `nk.FriendsDelete`, then a persistent notification with **code 1**, subject `"friends"`, content `{"title":"Friend Removed","message":"Your friend has removed you from their friends list"}` to every id in `user_ids`. Response `{"success":true,"message":"Friend Removed","data":{"success":true,"message":"Friend removed successfully"}}`.

Everything else in the social README (`social_*`) is commented out. See [[social]].

## Notifications module (`server:modules/notifications/init.go`)

| RPC | Registered | Handler | Request | Response |
|---|---|---|---|---|
| `notifications_list` | `init.go:15` | `rpc.go:13` | `{"limit":20,"cacheable_cursor":""}` (limit outside 1..100 → 10) | `{"success","message","notifications":[{"id","subject","content","code","sender_id","create_time","persistent"}],"cacheable_cursor"}` |
| `notifications_delete` | `init.go:20` | `rpc.go:65` | `{"notification_ids":["…"]}` | `{"success":true,"message":"Notifications deleted successfully"}`; scoped to caller |

Errors: `{"success":false,"error":"request_failed","message":"..."}`. There is deliberately no `notifications_send` RPC (`init.go:12-13`). The client uses the SDK's `ListNotificationsAsync`/`DeleteNotificationsAsync` instead. See [[notifications]].

## Playstyle module (`server:modules/playstyle/init.go`, no client caller)

| RPC | Line | Request | Response |
|---|---|---|---|
| `create_playstyle` | `init.go:9`, `rpc.go:12` | `{"name","slots":[{"slot_number":1,"spell_id":"x"|null}]}` | `{"playstyle_id","name","slots","success","message"}` |
| `get_playstyles` | `init.go:27`, `rpc.go:49` | ignored | `{"playstyles":[…],"success","message"}` |
| `update_playstyle` | `init.go:21`, `rpc.go:86` | `{"playstyle_id","name","slots"}` | as create; ownership checked |
| `delete_playstyle` | `init.go:15`, `rpc.go:136` | `{"playstyle_id"}` | `{"playstyle_id","success","message"}` |

Slot limit comment says "up to 7" (`types.go:10`); validation is in `validate.go` (not verified here).

## `healthcheck`
- `server:modules/healthcheck/init.go:9`, handler `healthcheck.go:11`. Returns `{"status":"ok","message":"Server is healthy"}`.

## Nakama hooks (not RPCs)
- `RegisterBefore/AfterAuthenticateEmail|Google|Apple|GameCenter` — logging only (`server:modules/auth/init.go:9-46`). See [[google-auth]].
- `RegisterMatchmakerMatched`, `RegisterBeforeRt("MatchmakerAdd")` — see [[matchmaking]].

## Source of truth in code
- `server:modules/main.go` — module registration order (race before character).
- `server:modules/*/init.go` — RPC ids; `server:modules/*/rpc.go`, `character/details.go`, `character/tutorial.go` — handlers.
- `server:modules/character/types.go`, `spellbook/types.go`, `spell_system/rpc_types.go`, `race/types.go`, `social/types.go`, `notifications/types.go`, `common/types.go` — wire structs.
- `server:modules/spell_system/version.go` — version stamp; `server:data/spells/*.yaml` — catalog.
- `client:Application/Modules/**/*Handler.cs`, `client:Application/Tutorial/TutorialSession.cs`, `client:Application/ArcaneDuel/*/MatchManager.cs` — every `RpcAsync` caller.
- `client:Core/Match/DuelV2.cs` — client-side version gate.
