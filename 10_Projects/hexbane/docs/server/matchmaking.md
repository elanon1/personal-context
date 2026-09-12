---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, server, matchmaking, nakama]
sources: ["server:docs/match/matchmaking.md", "server:docs/match/api-reference.md", "server:docs/API-REFERENCE-v2.md"]
---

# Matchmaking and match creation

Both handlers run the shared engine at 10 ticks/s. Current normal fallback contract: [[fallback-opponents]].

| Handler | Queue | Creation |
|---|---|---|
| `normal` | normal, custom queue enabled | queue RPCs create either two-human or human+persona assignments |
| `normal` | normal, custom queue disabled | Nakama matchmaker |
| `normal` | ranked | Nakama matchmaker, level ≥30 required |
| `ai_duel` | explicit training | `create_ai_arcane_duel` |

Built-in tickets use `queue=normal|ranked` (omitted means normal), forced 2-player counts and server query `+properties.queue:<queue>`. Unknown/mixed queues fail. Custom-enabled servers reject normal built-in tickets, including stale matched cohorts. Ranked stays separate. Normal match initialization reads its invitation allowlist; socket admission rejects outsiders and synthetic IDs.

The new queue selects a fallback deadline in 15–30 s, gives compatible humans priority before reservation, and presents one common match-found/accept flow. Details of reservation, leases and reconnect are in [[fallback-opponents]] and RPC shapes in [[rpcs]]. Ordinary human setup loads the current character, stats-derived combat profile, owned collection and race-specific slots; no fixed 200HP/100mana override.

Explicit `ai_duel` still uses `0000` / `Bot` and its distinct training route. It starts with standard spells only and drafts optional owned spells during the lobby; it no longer starts with a prefilled optional deck and appends a second copy. Its legacy slot copying is not used by fallback personas.

Reconnect replaces the old presence; a late leave from the old session cannot evict the new one. Normal precombat leave cancels with opcode9/opponent_left; combat disconnect does not itself end the match. Empty-match reaper: 30 s before first human, 5 s after all humans disappear. Settlement pending prevents premature termination. Normal assignment/persona leases are released on loop completion/termination; PvP never heartbeats a nonexistent persona lease.

`decline_match` authenticates the sender and signals `{kind:"decline",user_id}`; only an invited/current participant may decline before combat. Raw unauthenticated decline signals cannot cancel combat. Custom queues use `queue_decline`.

Client queue polling follows the current socket and account. Accept retries handle replaced transports; result delivery is de-duplicated and can be recovered from an authorized persisted receipt. Tests cover queue races and both complete local flows; rendered Godot/device playtests are separate.

## Source of truth in code
- server: `modules/match/normal_match`, `modules/match/ai_match`, `modules/matchmaking`, `modules/match/engine/core`
- client: `Application/ArcaneDuel/Normal`, `Application/ArcaneDuel/Bot/BotMatchManager.cs`

## Character already in a match (2026-09-12)

Both handlers run a read-only availability check before admitting a new player. `character_in_match` rejects a new match without removing the old lease or marking the queued player joined. The final atomic acquisition still guards races; its busy failure is opcode 9 with the same reason. Same-match reconnect remains allowed. See [[op_09_match_canceled]] for the client wait message. Fallback remains enabled at 15–30 seconds.
