---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, rpc, nakama]
sources: ["server:RPCs.md", "server:docs/API-REFERENCE-v2.md", "server:docs/match/api-reference.md", "server:docs/progression/client/menu-rpc-requirements.md", "client:docs/Server/progression/menu-rpc-requirements.md", "server:docs/client/client-implementation-prompt.md"]
---

# RPC reference (verified against code, 2026-09-07)

Every RPC the Nakama plugin registers, with the exact JSON it accepts and returns. All handlers are
registered from `server:modules/main.go` via each module's `InitModule`. Unless stated otherwise:

- **Auth**: Nakama runs RPCs with a session token; every handler reads `RUNTIME_CTX_USER_ID`. The only
  handlers that do not touch the user id are `healthcheck`, `get_races`, `get_race`, `get_starter_spells`,
  `get_entry_spells`, `get_spell`, `get_spell_lore`, `get_spell_details_yaml`.
- **Errors**: handlers return HTTP 200 with `{"success": false, "message": "..."}` (or `"error"` for
  `learn_spell`). Only `tutorial` and the two match RPCs return real gRPC errors. The gRPC error-code
  table in the old API-REFERENCE-v2 does not describe any other RPC.
- **Version stamp**: spell RPCs and `get_character_details`/`tutorial` embed
  `{"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2"}`
  (`server:modules/spell_system/version.go:3-19`). The client rejects other values
  (`client:Core/Match/DuelV2.cs:8-12`) in `get_player_spells` and `tutorial` only.

## Client ↔ server coverage

| Client calls | Server registers | Status |
|---|---|---|
| `create_character`, `get_my_character`, `get_character_by_id`, `allocate_stat_points`, `get_character_details`, `get_progression`, `tutorial`, `set_tutorial_completed`, `get_starter_spells`, `get_player_spells`, `get_spellbook`, `get_spell`, `learn_spell`, `get_races`, `find_friend`, `remove_friend`, `decline_match`, `create_ai_arcane_duel`, `create_character_match_story` | yes | OK (see per-RPC notes; `create_character_match_story` is broken server-side) |
| `get_users` (`client:Application/Modules/Social/Queries/GetUsers/GetUsersQueryHandler.cs:44`) | **no** | dead client code, query never dispatched |
| `start_story` (`client:Application/Modules/Story/Query/StartStory/GetSpellQueryHandler.cs:36`) | **no** (commented out in `server:modules/endless_story/init.go:13`) | dead client code |
| — | `debug_create_character`, `get_available_spells`, `get_entry_spells`, `get_my_spells`, `get_spell_lore`, `get_spell_details_yaml`, `get_race`, `create_playstyle`, `get_playstyles`, `update_playstyle`, `delete_playstyle`, `healthcheck`, `notifications_list`, `notifications_delete` | server-only (client uses Nakama SDK built-ins for friends/notifications) |

`RpcAsync` on the socket is used for `decline_match`, `create_ai_arcane_duel`, `create_character_match_story`; everything else goes through `client.RpcAsync(session, id, payload)`.

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
  `progression.CreationPoints` = 400, each effective (base + race modifier) ≥ 10, race floors/ceilings
  (`ValidateEffectiveStatLimits`, 0 = no limit), each base ≥ 1 → `spell_ids`: exactly
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
  remaining at that step and the new effective stat must stay inside the race floor/ceiling. The total
  does **not** have to equal all unspent points (that check is commented out, `rpc.go:231-235`).
  Row is locked `FOR UPDATE` in a transaction (`rpc.go:204-270`).
- Response `{"character": <Character>|null, "message", "success"}`. Errors: `"All stat values must be non-negative"`, `"not enough stat points"`, `"invalid point allocation"`, `"<stat> may not exceed N for race X, got V"`, `"<stat> requires at least N for race X, got V"`.

### `get_character_details`
- `init.go:40`, handler `details.go:156`. Client: `GetCharacterDetailsQueryHandler.cs:26-28`. Contract in [[character-details]].

### `get_progression`
- `init.go:54`, handler `rpc.go:320`. Client: `GetProgressionQueryHandler.cs:28`.
- Request ignored. Response:
  ```json
  {"success":true,"message":"Progression retrieved successfully","level":1,"experience":0,
   "experience_to_next_level":100,"magic_points":0,"magic_points_spent":0,"available_magic_points":0,
   "spell_slots":3,"next_spell_slot_level":4,"unspent_stat_points":0}
  ```
- **Stale maths inside this handler**: `experience_to_next_level` uses `100·1.5^(level−1)` and
  `next_spell_slot_level` uses `{4,6,10,15,20,25}` (`rpc.go:341-357`), while the progression module
  uses `100·1.5^(level−2)` (`server:modules/progression/xp.go:10-18`) and unlock levels `{4,8,12}`
  (`constants.go:33`). `get_character_details` uses the correct module functions. Prefer it.

### `tutorial` and `set_tutorial_completed`
- `init.go:45` / `init.go:49`; handlers `tutorial.go:106` / `rpc.go:153`. Contract in [[server-tutorial]].
- `set_tutorial_completed` always returns `{"character":null,"message":"Use tutorial progression; tutorial state cannot be reset","success":false}`. Client caller `SetTutorialCompletedCommandHandler.cs:31` is only reachable through `CharacterService.SetTutorialCompleted`, which nothing calls.

## Spellbook module (`server:modules/spellbook/init.go`)

### `get_starter_spells`
- `init.go:33`, handler `rpc.go:375`. Client: `client:Application/Modules/Spell/Queries/GetEntrySpells/GetEntrySpellsQueryHandler.cs:39`.
- Request ignored. Returns every catalog spell with `starter: true` (6 in `duel_v2.2`: barrier, cleanse, firebolt, heavy_bolt, mend, poison):
  ```json
  {"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2","success":true,
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
  {"combat_protocol":2,"ruleset_id":"duel_v2","catalog_version":"duel_v2.2","success":true,"message":"...",
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

## Match RPCs

### `create_ai_arcane_duel`
- `server:modules/match/ai_match/init.go:34`. Client: `client:Application/ArcaneDuel/Bot/BotMatchManager.cs:92` (socket RPC).
- Creates an `ai_duel` match; response `{"success":true,"message":"Match created","data":{"match_id":"<id>.nakama1"}}`; failure is a gRPC error from `MatchCreate`. See [[matchmaking]].

### `decline_match`
- `server:modules/match/normal_match/init.go:31-53`. Client: `client:Application/ArcaneDuel/Normal/MatchManager.cs:158-159`.
- `{"match_id":"..."}` → signals the match with `"decline"`; response `{"success":true,"message":"Match declined","data":null}`. gRPC errors `match_id is required`, `failed to decline match`. See [[matchmaking]].

### `create_character_match_story` (broken)
- `server:modules/endless_story/create_character/init.go:30`. Client: `client:Application/EndlessStory/CreateCharacter/MatchManager.cs:60` (registered as keyed `IMatchManager` `"create_character"`, `client:Game/DI/ServiceBootstrapper.cs:109`, never resolved).
- Calls `nk.MatchCreate(ctx, "create_character", …)` but the match handler is registered as `"v2_create_character"` (`init.go:25`), so the call fails at runtime. Story RPCs `start_story`, `create_character_story`, `continue_character_story` are commented out (`server:modules/endless_story/init.go:13-23`).

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
