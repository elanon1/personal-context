---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, matchmaking, nakama]
sources: ["server:docs/match/matchmaking.md", "server:docs/match/api-reference.md", "server:docs/API-REFERENCE-v2.md"]
---

# Matchmaking and match creation

Two authoritative match handlers exist; both use the same engine (`server:modules/match/engine/`) and
run at `state.TicksPerSecond = 10` (100 ms ticks, `server:modules/match/engine/state/state.go:12`).

| Handler name | Registered | Mode | Created by |
|---|---|---|---|
| `normal` | `server:modules/match/normal_match/init.go:61` | `MatchModeNormal` | matchmaker (`MakeMatch`) |
| `ai_duel` | `server:modules/match/ai_match/init.go:27` | `MatchModeAI` | `create_ai_arcane_duel` RPC |

A third handler, `v2_create_character` (`server:modules/endless_story/create_character/init.go:25`), is the
story prototype; its RPC creates `"create_character"` and therefore fails. Not part of the game flow.

## PvP (`normal`)

1. Client queues: `socket.AddMatchmakerAsync("", 2, 2)` (`client:Application/ArcaneDuel/Normal/MatchManager.cs:89-90`), no properties.
2. `BeforeMatchmakerAdd` (`server:modules/match/normal_match/matchmaker.go:34-48`, registered as
   `RegisterBeforeRt("MatchmakerAdd")` at `init.go:26`) forces `MinCount = 2`, `MaxCount = 2`,
   `Query = ""` on every request. Client-supplied query and counts are ignored. No properties are used.
3. `MakeMatch` (`matchmaker.go:15-32`, `RegisterMatchmakerMatched` at `init.go:21`) logs the entries and
   calls `nk.MatchCreate(ctx, "normal", {"invited": entries})`. The `invited` param is not read by
   `MatchInit` (`init.go:71-80`).
4. Both clients receive `ReceivedMatchmakerMatched`; the client shows **Match found** and waits for the
   player (`MatchManager.cs:132-137`).
   - Accept → `JoinMatchAsync(matchId)` (`MatchManager.cs:139-149`).
   - Decline → RPC `decline_match {"match_id"}` (`MatchManager.cs:151-167`) → `nk.MatchSignal(match, "decline")`
     (`init.go:31-53`) → `MatchSignal` broadcasts opcode **8** `OP_MATCH_DECLINED` `{}` to everyone in the
     match and sets `TerminateMatch` (`signal.go:15-31`).
5. `MatchJoinAttempt` (`join.go:54-70`): existing participants always allowed (rejoin); otherwise
   rejected with `"Match is full."` once two players are in.
6. `MatchJoin` (`join.go:15-52`): new player → `core.BuildPlayerState` (character + learned spells +
   both standard spells, 200 HP / 100 mana, draft slots capped to learned count,
   `server:modules/match/engine/core/player_setup.go:18-58`). On failure the match broadcasts opcode **9**
   `OP_MATCH_CANCELED` `{"reason":"player_setup_failed"}` and terminates. When the second player has
   joined the match transitions to `PhaseConnecting`.

## Bot duel (`ai_duel`)

1. Client calls `create_ai_arcane_duel` over the socket (`client:Application/ArcaneDuel/Bot/BotMatchManager.cs:92`),
   gets `{"success":true,"message":"Match created","data":{"match_id":"…"}}`, shows **Match found**,
   then joins on accept (`BotMatchManager.cs:114-124`). Decline is client-side only (no RPC); the empty
   match is reaped by the server (see below).
2. `MatchInit` (`server:modules/match/ai_match/init.go:53-64`) seeds the bot as player `"0000"`
   (`bot.go:16`), level 1, random race from the registry (`bot.go:53-59`).
3. `MatchJoinAttempt` (`join.go:74-87`): one human only (`"Only one player allowed in this match."`);
   rejoin of that human allowed.
4. `MatchJoin` (`join.go:17-72`): on the human's first join the bot's `SpellSlots`/`MaxSpellSlots` are
   copied from the human and the bot drafts that many random non-standard spells
   (`spellbook.GetRandomSpells`, `server:modules/spellbook/db.go:70-93`) plus the standards; then `PhaseConnecting`.

## Rejoin / reconnect

- Server: `core.RestorePresence` (`server:modules/match/engine/core/rejoin.go:11-27`) — if the user id is
  already in `Players`, the new presence replaces the old one and, when the phase supports snapshots
  (combat), a fresh **opcode 32** snapshot is sent to that presence only.
- `MatchLeave` only drops a presence whose session id matches the stored one (`normal_match/leave.go:21-30`,
  `ai_match/leave.go:13-17`), so a replacement socket is not evicted by the old socket's leave.
- Client: on `GameEvents.ConnectionEstablished` both managers call `JoinMatchAsync(MatchContext.MatchId)`
  again if a match is in progress and not over (`MatchManager.cs:72-80`, `BotMatchManager.cs:72-80`);
  `"local-training"` (tutorial) is skipped. Failure raises `OnMatchRestoreFailed`.

## Leaving and termination

- PvP: a presence leaving during `connecting`, `lobby_picking`, `lobby_countdown`, `loading`,
  `game_countdown` cancels the match: remaining players get opcode **9** `{"reason":"opponent_left"}` and
  the match terminates (`normal_match/leave.go:32-53`). Leaving during combat does not end the match.
- AI: leaving only removes the presence (`ai_match/leave.go`).
- Empty-match reaper (`server:modules/match/engine/core/loop.go:73-92`): a match nobody has joined yet
  is terminated after `JoinGraceTicks` = 30 s; a match that had players and is now empty after
  `EmptyMatchTerminateTicks` = 5 s (`loop.go:14,24`). `TerminateMatch` ends the loop on the next tick.
- Client `LeaveMatch` → `LeaveMatchAsync` (both managers).

## What the old docs got wrong

- Match handler name was `"lobby"`, query `"*"`: now `"normal"` and `""`.
- `TicksPerSecond`, `GameTime = 300`, `playerDataSyncInterval`: only `TicksPerSecond = 10` still exists in this form; combat timing lives in `duel_v2` (see [[combat-v2]]).
- No matchmaker properties or skill/region queries are used anywhere.

## Source of truth in code
- `server:modules/match/normal_match/{init,matchmaker,join,leave,signal}.go` — PvP flow, `decline_match`.
- `server:modules/match/ai_match/{init,join,leave,bot}.go` — bot flow, `create_ai_arcane_duel`.
- `server:modules/match/engine/core/{rejoin,player_setup,loop}.go` — rejoin snapshot, player build, reaper.
- `server:modules/match/engine/match_types/op_codes.go` — `OP_MATCH_DECLINED = 8`, `OP_MATCH_CANCELED = 9`, `OpCombatSnapshot = 32`.
- `server:modules/match/engine/state/state.go` — tick rate, match modes.
- `client:Application/ArcaneDuel/Normal/MatchManager.cs`, `client:Application/ArcaneDuel/Bot/BotMatchManager.cs` — client side; keyed DI `ad_normal` / `ad_ai` (`client:Game/DI/ServiceBootstrapper.cs:107-108`).
