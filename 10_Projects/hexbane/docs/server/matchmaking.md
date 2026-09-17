---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-17
verified: 2026-09-17
tags: [hexbane, server, matchmaking, nakama]
sources: ["server:docs/match/matchmaking.md", "server:docs/match/api-reference.md", "server:docs/API-REFERENCE-v2.md"]
---

# Matchmaking and match creation

Both handlers run the shared engine at 10 ticks/s. Current normal fallback contract: [[fallback-opponents]].

| Handler | Queue | Creation |
|---|---|---|
| `normal` | normal, custom queue enabled | queue RPCs create either two-human or human+persona assignments |
| `normal` | normal, custom queue disabled | Nakama matchmaker |
| `normal` | ranked (disabled) | New tickets and matched cohorts are rejected |
| `ai_duel` | explicit training | `create_ai_arcane_duel` |

Built-in tickets accept `queue=normal` (omitted means normal), forced 2-player counts and server query `+properties.queue:normal`. Ranked requests fail with `queue unavailable` (code9), including stale matched cohorts. Unknown/mixed queues fail. Custom-enabled servers reject normal built-in tickets, including stale matched cohorts. Existing match invitation/join guards remain for already-created matches; no new ranked queue is exposed (HEX-20). Normal match initialization reads its invitation allowlist; socket admission rejects outsiders and synthetic IDs.

The new queue selects a fallback deadline in 10–20 s, gives waiting humans priority before reservation, and currently ignores character-level differences when pairing two humans. If no human opponent is available past the deadline, the existing fallback persona is still eligible. It presents one common match-found/accept flow. Details of reservation, leases and reconnect are in [[fallback-opponents]] and RPC shapes in [[rpcs]]. Ordinary human setup loads the current character, stats-derived combat profile, owned collection and race-specific slots; no fixed 200HP/100mana override.

Explicit `ai_duel` uses synthetic ID `0000`, wire name `AI · Level N` (the client displays `AI`) and difficulty1–5 selected through `create_ai_arcane_duel`. Its draft waits0.3–0.6s per pick (up to0.7s including scheduling), independently of combat difficulty. It starts with standard spells only and drafts optional owned spells during the lobby; it no longer starts with a prefilled optional deck and appends a second copy. Its legacy slot copying is not used by fallback personas.

Reconnect replaces the old presence; a late leave from the old session cannot evict the new one. Normal precombat leave cancels with opcode9/opponent_left; combat disconnect does not itself end the match. Empty-match reaper: 30 s before first human, 5 s after all humans disappear. Settlement pending prevents premature termination. Normal assignment/persona leases are released on loop completion/termination; PvP never heartbeats a nonexistent persona lease.

`decline_match` authenticates the sender and signals `{kind:"decline",user_id}`; only an invited/current participant may decline before combat. Raw unauthenticated decline signals cannot cancel combat. Custom queues use `queue_decline`.

Client queue polling follows the current socket and account. Accept retries handle replaced transports; result delivery is de-duplicated and can be recovered from an authorized persisted receipt. Tests cover queue races and both complete local flows; rendered Godot/device playtests are separate.

## Source of truth in code
- server: `modules/match/normal_match`, `modules/match/ai_match`, `modules/matchmaking`, `modules/match/engine/core`
- client: `Application/ArcaneDuel/Normal`, `Application/ArcaneDuel/Bot/BotMatchManager.cs`

## Character already in a match (2026-09-12)

Both handlers run a read-only availability check before admitting a new player. `character_in_match` rejects a new match without removing the old lease or marking the queued player joined. The final atomic acquisition still guards races; its busy failure is opcode 9 with the same reason. Same-match reconnect remains allowed. See [[op_09_match_canceled]] for the client wait message. Fallback remains enabled at 10–20 seconds.

## Explicit AI difficulty (HEX-21, 2026-09-15)

Training uses `bot_ai.TrainingProfile` selected only for `MatchModeAI`; natural fallback profiles remain separate. Levels1–5 are Beginner, Easy, Normal, Hard, Expert. Reaction/decision latency, choice temperature, planned-queue probability and mistake chance vary; no hidden enemy state or stat/damage multiplier is granted. Level1 reacts in1.5–2.5s plus0.8–1.5s decision delay, makes frequent mistakes and does not queue during recovery. Level5 reacts in0.1–0.2s plus0–0.1s decision delay, frequently plans ahead and uses the existing visible-cast/impact/defense scorer. A response with zero configured latency still submits on a later engine tick.

As of 2026-09-16, client vs AI starts directly with moderate difficulty 3 (Normal), serialized through the generated JSON context. No difficulty selector or level label is displayed. The server still supports all five profiles. Invalid RPC choices fail without creating a match. Existing AI build/stat/slot rules remain; difficulty changes decision behavior.

### HEX-21 verification (2026-09-15)

Full Go suite and Linux Nakama plugin build pass. A disposable Nakama 3.27/Postgres instance validated all five difficulty RPCs, invalid request rejection, opponent labels, four fast AI draft picks and the first combat snapshot. Maximum observed per-pick wait was 706 ms. Live verification exposed an unpaired Consume Venom draft stall: combo scoring now falls back to an owned legal spell when no preferred candidate exists, covered by a regression that failed before the fix.

The deterministic equal-build calibration ran 240 duels (30 seeds, both seats, level 5 versus each lower level): expert W/L/D = 31/2/27 against 1, 17/10/33 against 2, 23/2/35 against 3, 18/3/39 against 4. This establishes profile differences, not subjective human difficulty. Client build and 174 reflection-disabled JSON checks pass; five-level selector rendered at 960x540 and 960x432 with no measured overflow. Production deployment and physical-device play were not performed.


## Friend matches from invitations (HEX-30, 2026-09-17)

`duel_invite_reply` (accept) calls `nk.MatchCreate("normal", {"friend_users": "[\"<a>\",\"<b>\"]"})`. `MatchInit` (`server:modules/match/normal_match/init.go`) parses that JSON array, rejects anything but two distinct non-empty ids (the match is then not created) and seeds `InvitedUsers` with them, so `admission.go` admits only those two accounts exactly like a matchmaker-invited pair; no queue ticket, no fallback persona, not ranked. Both selected characters already hold a 2-minute `character_match_locks` row written by the accept transaction, and the ordinary acquisition on join extends it. The custom queue's `Join` now also locks the `users` row first and reads the **selected** character's level, so a queue ticket and a character switch cannot interleave.
