---
type: project
project: Hexbane
area: plans
status: proposed
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, matchmaking, ai, design]
sources: ["server:modules/match", "server:data/spells", "client:Application/ArcaneDuel/Normal/MatchManager.cs"]
---
# Natural fallback opponent — design

> Proposal, not implemented or approved for deployment. `verified` means the baseline was inspected on this date, including uncommitted combat restoration changes. Timing, probabilities and budgets below are initial tuning hypotheses, not measured player behaviour. Implementation plan: [[2026-09-08-fallback-player-plan]].

## Outcome and scope

A player searching for a normal duel should receive a plausible opponent after a bounded wait when another human is unavailable. Prefer human pairings. Use the same match-found, acceptance, draft, arena and result presentation. The opponent has a stable fictional identity, a legal character build, coherent preferences and imperfect execution. Do not show a dedicated bot label or use the explicit training flow for this fallback.

Default scope: normal unranked queue, ordinary current XP/daily bonuses and skill gains for the human, no persistence of bot rewards. Ranked fallback is disabled. These are proposed defaults pending product review. The user requested a design and implementation plan only; this session changes documentation only.

“Nothing indicates AI” is a presentation and behavioural target, not a guarantee of undetectability. A server-controlled participant has no human socket presence; packet inspection or repeated statistical analysis can reveal that. Do not fabricate network sessions, social conversations, human account history or online-population counts. Keep truthful internal bot attribution for support, analytics and result accounting. Social/profile functionality must be coherent before enabling fallback on a screen that exposes it.

## Verified starting point

- `normal_match/matchmaker.go` forces count 2 and empty query; `MakeMatch` creates `normal`. `MatchInit` ignores its `invited` parameter; new joins currently use capacity checks rather than an invitation allowlist.
- The normal client uses `AddMatchmakerAsync`, waits for `ReceivedMatchmakerMatched`, and joins only after acceptance. There is no queue-age authority or fallback transaction.
- `ai_match` already uses the shared 10 Hz engine, creates `0000` / `Bot` and a random race, models level 1, and copies the human's draft slot count. This can produce a race/level/slot combination that differs from legal progression.
- The bot currently starts with random drafted spells already in `SelectedSpells`; the lobby separately records visible selections. The new flow must construct the final loadout from the actual completed draft.
- Lobby bot picking happens immediately on its turn and has its own mutation path. The timer is **35 seconds total per player across their turns**, not 35 seconds per pick; the timer is not reset after each choice.
- Connecting auto-readies bots. Loading currently broadcasts game-ready then advances on a tick; it is not a per-player asset-loading acknowledgement system.
- `game/ai.go` runs every fourth tick, has three fixed priority policies, a 400 ms visibility check on selected effects/casts, and uses `GamePhaseState.Submit`. It still receives full player objects; its public-only restriction is a convention, not an enforced API boundary.
- `PublicPlayerView` / `PrivatePlayerView` do not expose `IsBot`. Game-over skips persistence for bots. Current combat profile and skill gain restoration must be retained.

## Approaches considered

| Approach | Benefit | Cost / reason |
|---|---|---|
| Client timeout then `create_ai_arcane_duel` | Small prototype | Client controls fallback age; cancellation races human pairing; separate transport/UI branches; not recommended for release |
| Server-owned queue + shared engine + utility AI | Atomic assignment, uniform flow, reproducible decisions | More queue work and migrations; recommended |
| Real synthetic Nakama accounts with client bots / learned policy or LLM | Closer transport shape; potential broader behaviours | Sessions, moderation/profile lifecycle, inference cost and unpredictable latency; unnecessary for 14 spells |

Keep Nakama for authoritative matches. Own assignment for the new normal queue, with PostgreSQL as the arbiter. Do not run this queue and the built-in matchmaker concurrently for the same queue cohort. Heroic Labs documents the normal matchmaker callback/authoritative match creation model; the assignment service here is an application-level design, not a claimed built-in fallback API: [Matchmaker documentation](https://heroiclabs.com/docs/nakama/concepts/multiplayer/matchmaker/). The vendored `NakamaModule` has `MatchCreate`; no server `MatchmakerRemove` method was found, so the design does not assume one.

## 1. Queue and assignment authority

### Client contract (new RPCs, proposed)

All use the existing response envelope. Identity comes from authenticated context; never accept user ID, bot flag, level or elapsed wait from the request.

| RPC | Request | Response data |
|---|---|---|
| `queue_join` | `{request_id: UUID, mode: "normal", protocol: 2}` | `{queue_id, generation, state, poll_after_ms: 1000}` |
| `queue_status` | `{queue_id, generation}` | `{queue_id, generation, state, assignment_id?, match_id?, expires_at?}` |
| `queue_cancel` | `{queue_id, generation}` | `{state: "canceled"}` or current terminal state |
| `queue_accept` | `{assignment_id}` | `{state, match_id?, expires_at?}` |
| `queue_decline` | `{assignment_id}` | `{state: "declined"}` or terminal state |

States: `searching → reserved → offered → accepted → joined → completed`; cancellation, decline, expiry and failed allocation are explicit branches. A canceled generation never becomes active again. `queue_status` doubles as a heartbeat while searching. Both PvP and fallback produce the same offer DTO, without a bot-specific route/field. Private assignment storage records `opponent_kind`.

Poll every 1 s while searching/reserved/offered; serialize polls; discard responses from old socket/account/queue/generation. RPC failure retries use bounded backoff up to 3 s; UI can cancel or retry. A server lease of 10 s marks abandoned searching tickets unavailable. Reconnection resumes a live generation; after lease expiry it creates a new one with a new wait. Returning an existing match does not restart queueing.

### Matching and timing

1. `queue_join` derives an eligibility/build summary from the real character and stores a DB timestamp plus one sampled fallback deadline: triangular 35/45/55 s (min/mode/max). No exact recurring timeout, client clock or per-poll resampling.
2. Each live status/join request gives the coordinator an opportunity to assign the oldest compatible tickets. MVP compatibility: normal mode, protocol, server region and level distance ≤2, widening to ≤5 after 20 s; skills/build strength inform future calibration. No invented MMR exists today. Different races can legitimately have different slots.
3. Under a transaction-scoped advisory lock per compatibility pool, select live searching tickets in oldest-first order. Always try human pairing first. If none is compatible and a ticket has passed its deadline, reserve it for fallback. A simultaneous human arrival after reservation does not replace an opponent already reserved.
4. Transition the one or two rows to `reserved`, assign a shared assignment UUID and allocation generation, record a 10 s allocation lease; commit. One active row per user via a partial unique index. All cancellation/allocation state transitions use the same pool lock, consistent row order and compare-and-swap predicates.
5. Create the authoritative match **outside** the SQL transaction. Publish the offer only if the assignment/generation remains reserved and valid. Both human tickets receive the same published match ID. Polls recover delivery if a response was lost.
6. On crash or timeout, expire the allocation lease before retrying with a new allocation generation. A late creator may not publish. Matches reject joining until their exact ID/generation is the published assignment; unused instances terminate via the existing empty-match reaper. This guarantees one valid published assignment, not exactly one physical `MatchCreate` call under failure.
7. `queue_cancel` before acceptance invalidates any reservation/offer and notifies the counterpart via its next status read. Release a still-searching counterpart into a new generation while preserving its original wait age; explicit cancellation resets age. An accepted assignment uses decline/leave semantics, not silent queue resurrection.
8. Offers expire 20 s after publication; synthetic acceptance occurs 1.2–4.5 s after publication, clamped within the offer deadline. A human accepting earlier can join and see the ordinary connecting screen; the bot becomes ready only after its acceptance deadline. PvP still waits for both humans. The match's existing 30 s join grace must cover the offer and readiness window.
9. `queue_accept` is idempotent per user/assignment. Allowlist only the assigned humans; a synthetic ID can never socket-join. Server validates accepted, unexpired admission for first join; an established participant can reconnect under current session replacement rules.
10. Expiry and cleanup run opportunistically in queue operations; no sleeping goroutine per queued user. Old terminal rows are removed in bounded batches by a scheduled maintenance job. A stopped client cannot spawn fallback because its lease becomes ineligible.

Limit queue mutation RPCs per user; reject parallel join generations. Gate legacy `MatchmakerAdd` for the new queue cohort so custom and built-in assignments cannot compete. Old clients stay on an explicitly isolated legacy cohort during local compatibility testing; do not claim cross-cohort matchmaking. Kill switch stops new fallback reservations, keeps human pairing and already published matches working.

## 2. Fictional identity and legal build

Store bot personas in `bot_personas`: random UUID, unique normalized nickname, race, legal base-stat allocation, level band, learned spell IDs, skill values, style seed, version. This is an application identity, not a Nakama login. A match participant uses that stable UUID; reserve a persona against concurrent matches and release its lease at completion/expiry. Prefer a different persona from the human's last 20 fallback opponents or last 7 days; if pool exhaustion requires a repeat, keep the same identity/build rather than renaming mid-session.

Generate identities from a curated base-name dictionary and weighted formats, validate 3–20 characters with the character-name rules, normalize for uniqueness and reject reserved/offensive/confusable names and collisions with existing character names. Do not copy existing players. Retry up to 20 times, then use a prevalidated spare pool; failure disables this allocation and leaves the user searching.

Initial name format weights (sum 100%): plain 50%, numeric suffix 22%, country suffix 12%, separator + numeric 8%, prefix/casing variant 6%, country + digits 2%. Examples: `Velrin`, `Kora83`, `Nox_PL`, `Aster_7`, `xMiren`, `RavikDE24`. Country-like suffixes are cosmetic nickname text, not asserted geolocation. Avoid every name having a suffix or all numeric suffixes resembling years. Use thousands of combinations; evaluate distribution and repeats, not merely random character count.

Persona identity and build are stable; session mood and timings vary. Skill tier is selected before the match using coarse human level/experience bands, never hidden future actions. Build generation must obey the same race limits, 400 creation points + earned level-up allocation, actual starter/MP ownership budget, skill caps and progression slot ladder. Derive effective costs, cast times, HP/mana and damage through existing combat code. Do not copy the opponent's exact deck or slot count. Unknown race/catalog/profile version fails validation, rather than creating an impossible fallback race.

Internal profile lookup returns the stable persona when a match-authorized player inspects that participant; expose only real supported display fields. Do not invent match counts, last-seen human activity or online status. Before release audit friend/chat/report/profile actions: reports route internally; unsupported social actions must use a consistent product policy for all match opponents. If a screen depends on real Nakama-user lookup, either implement an application profile abstraction or keep fallback disabled there. Social impersonation is not part of this subsystem.

## 3. Draft and lobby behaviour

Use a single validated `SelectSpell` path for humans and bots: correct turn, owned non-standard spell, unique selection, free slot. Fix the current missing explicit ownership guard as part of this path. Start `SelectedSpells` with standards only; after draft finalize standards + actual picks, never all owned spells.

On each bot turn draw one deadline and remember a turn ordinal, not only user ID (the same player can receive consecutive picks). Typical first choice 2.5–6.5 s, later choices 1.2–4 s; 10% of eligible turns hesitate an extra 1–2.5 s. These are bounded by the remaining **total** timer:

`max_delay = max(0, remaining_seconds - 0.8 * (remaining_picks - 1) - 1.0)`

`delay = min(sampled_delay, max_delay)`

When exhausted, use the existing legal auto-fill. Never sleep in a match callback; schedule in 100 ms ticks, round upwards and invalidate on phase/turn change. Slow personas must not force a timeout every seven-slot draft. Rejoining a human must not reset a scheduled bot pick.

Draft score: role coverage + persona preference + synergy with own picks + limited response to already revealed enemy picks − redundant roles − mana burden. Example archetypes: pressure (`firebolt`, `heavy_bolt`, `poison`); venom (`poison`, `consume_venom`, sustain); control (`delayed_hex`, `paralysis`, `dispel`); sustain (`mend`/`greater_heal`, `barrier`, `regeneration`). These are preferences over owned spells, not fixed mandatory decks. Standards already supply a cheap attack and reflection. Never take Consume Venom without owned Poison and a slot plan to draft it; lower-level personas cannot own combinations they could not purchase.

Connecting readiness gets a bounded 0.7–2 s preparation delay after bot acceptance. Keep the existing shared game countdown; do not add a bot-only loading screen or multiply lobby delays with artificial slow loading. Loading itself stays shared because current code does not model separate load acknowledgements.

## 4. Decision architecture

Use a small utility scorer plus a finite-state intention layer; no LLM or online model training.

`public snapshot/event projection → delayed perception → intent → legal candidates → utility scores → weighted choice → scheduled command → normal Submit/validation`

- `bot_ai` owns plain immutable observation and decision types. It must not import `engine/state`, `phase/game`, DB or Nakama. The game adapter projects only fields that the opponent-facing protocol actually exposes plus the bot's own private state.
- Perception includes own resources/loadout/queue, visible enemy resources, released/revealed casts, public effects and impacts with observed timestamps. It excludes enemy pending/queued commands, unexposed spellbook, server RNG state, future dodge rolls and future actions. No direct pointer to enemy `PlayerState` crosses this boundary.
- Maintain a short history of public observations and expose each newly observed event only after sampled perception latency. This applies to HP/mana/status changes and immunity, not just cast starts. A fresh reaction cannot use data from before it was delivered by the 200 ms snapshot/event schedule. Own state can be current.
- Random streams are independently derived from a server-only match seed for persona, draft, timing, decisions and mistakes; combat RNG remains independent. Sort candidates and IDs before drawing. Store seed and AI/catalog/config versions server-side for replay; never in client payloads.
- Persistent intents: opening, pressure, set-up combo, defend, recover mana, finish. Hold an intent for 2–5 decisions unless a perceived emergency invalidates it. A plan has at most 2–3 actions and is revalidated after each step.
- Each evaluation considers at most 16 candidates including meditate, clear-queue and wait. Check every 100 ms only whether a decision is due; most ticks do no scoring. Allow planning during casting/recovery and use the ordinary single queued action. Do not auto-queue perfectly every time.

### Choosing an action

Hard eligibility: owned/selected spell, known command, alive, compatible status, affordable with current own effective cost. Predicted tactical success is a score, not an oracle. Recheck through `Submit`, then execution validation. A rejected or stale scheduled action is discarded/rescheduled without command spam.

Normalize score components to [−1,1]: expected damage over a short horizon, useful healing, prevented visible damage, control value, combo progress, mana efficiency, persona preference, repetition penalty and overkill/overheal/waste. Initial weights: damage 1.0, survival 1.3, combo 0.6, economy 0.5, persona 0.3, waste −1.0; tune by tier. Use stat-derived estimates and public observations, not raw YAML damage or future engine rolls.

Keep viable candidates within 0.25 of the best normalized final score, capped at top 3. Sample with `P(a) ∝ exp((U(a)-max(U))/temperature)`, temperature 0.12–0.30 per persona. Fixed-seed stable ordering makes this reproducible. A short memory penalizes repeating the same 3–4 action prefix, but does not forbid the best response or a necessary repeated basic attack.

### Spell-specific reasoning (all 14)

| Spell | Useful reasoning / typical imperfect execution |
|---|---|
| Magic Arrow | Cheap pressure, finish, consume enemy mirror with a low-cost package; sometimes overuses a familiar option |
| Mirror Reflection | Predicted hostile impact after own cast can finish, no active mirror; can react late or waste it on a small threat |
| Firebolt | Efficient pressure when mana reserve permits; varies with expected risk |
| Heavy Bolt | Damage window when enemy cannot visibly interrupt in time; reckless persona commits too often |
| Poison | Apply if not active; value denying meditation and regeneration as well as damage |
| Cleanse | Respect hex-before-poison removal; prioritize by imminent harm and restoring mana/regen; sometimes delayed |
| Mend | Fast emergency recovery; estimate overheal and whether it resolves before visible lethal impact |
| Barrier | Absorb expected near-term damage/hex; does not block hostile statuses; never assume it stops paralysis |
| Delayed Hex | Pressure or set-up when absent; detonation is not reflected, application is; bait a cleanse before following up |
| Paralysis | Interrupt only if own effective cast resolves before the observed enemy cast ends; respect visible immunity; damage does not break paralysis |
| Greater Heal | Larger but slower sustain; less useful when it arrives too late |
| Regeneration | Long-horizon sustain while unpoisoned; no repeated refresh of an existing effect |
| Dispel | Respect actual priority mirror→shield→regeneration and its own reflectability; cannot simply remove a mirror safely |
| Consume Venom | Only own poison qualifies; weigh remaining poison/meditation denial versus immediate kill/burst; do not always consume immediately |

Meditation needs a valid unpoisoned, unparalyzed idle state and 800 ms ramp. Choose a resource target by intended combo/reserve, not a fixed universal 70%. A queued cast stops meditation; `clear_queue` does not cancel an active cast. Do not design dodge buttons, movement or cooldown logic absent from this game.

## 5. Human-like timing and mistakes

Each persona stores aggression, caution, combo commitment, mana reserve, favourite spell families, tempo and mistake tendencies. Each match adds small bounded mood variation. No continuous difficulty adjustment to force a victory/loss; skill parameters stay fixed after assignment.

| Tier | Perception + decision reaction, before tick rounding | Planned queue use | Deliberately suboptimal decision opportunities |
|---|---|---|---|
| Novice | 450–1100 ms | 25–45% | 12–18% |
| Regular | 300–750 ms | 45–70% | 6–10% |
| Experienced | 250–550 ms | 65–85% | 2–5% |

Draw bounded skewed distributions around a persona baseline, not a uniform delay every tick; visible-threat reactions cannot beat the tier minimum. Planned continuation may execute immediately after recovery because it was queued earlier. Add action-choice latency 100–250 ms to non-reactive idle deliberation; don't stack it again onto the reaction column above. Quantize all commands to the actual 100 ms tick.

Mistakes are contextual: tunnel vision for 1–2 decisions, overcommitting mana, choosing the second-best heal/attack, delaying cleanse, breaking a combo under pressure, meditating a little too long, failing to queue, defending against a minor cast. Choose from defined mistakes whose preconditions hold; otherwise skip the mistake draw. At most one mistake episode per 5 s; avoid chained intentional severe errors. Panic below a persona HP threshold can amplify its established tendency. Never implement random illegal spell IDs, mana cheats, impossible casts, scripted suicide or intentional server-level disconnects as “humanisation”.

Keep both recurrent habits and variation: a cautious persona regularly saves mana, but need not cast the same sequence in every match. A tactical mistake can still lose the match; do not secretly repair it with bonus stats or RNG overrides.

## 6. Operations, rewards and lifecycle

Configuration is versioned and validated at startup: fallback enabled (default false), 35/45/55 s deadline, 10 s search/allocation leases, 20 s offer TTL, persona pool limit, concurrent bot cap, supported protocol, tier weights, delay/error distributions. Invalid config disables new fallback with a clear server diagnostic. Preserve human queueing. Existing matches retain their copied config and AI version when switches change.

When capacity/persona allocation fails, remain searching and retry with bounded 2–5 s backoff; never promise a match by 55 s under outage/capacity exhaustion. Expose ordinary retry/cancel UI. The cap counts reserved and live personas. No deployment in this task; local Docker only under the current infra policy.

Persist human rewards exactly once per `(match_id, user_id)` through a result ledger and a transaction with character update. Log opponent kind separately so PvP metrics are not polluted. Bots earn no XP, skills, wins or ranking entries. Repeated AI farming is measured before introducing a separate reward rule; ordinary AI skill gains obey normal caps. Disconnect during combat keeps current combat/rejoin behaviour. At game over store the result before delivery; a new result lookup for the participant can recover a lost opcode 50, rather than duplicate rewards.

No human replacement by AI mid-fight in v1. Pre-combat human leave cancels under existing normal-match policy. Bot state, intention, seed and draft deadline remain in the live match across human reconnect; new socket sequence handling follows existing restore semantics. A whole match-process crash uses the ordinary match failure policy, not fabricated continuation.

## 7. Acceptance and measurement

Correctness gates:

- Concurrent join/status/cancel/fallback requests yield at most one valid assignment; old generations cannot publish or accept; strangers cannot join or decline another assignment; creation crash/late completion leaves only unjoinable reaped orphans.
- Every persona is legal for race, level, skills, spell budget and slot count. Identity remains stable on reconnect. Names are unique and format frequencies match configuration within statistical tolerance.
- No bot first-tick draft selection except genuine timer exhaustion; seven-slot draft respects 35 s aggregate budget; selections and final loadout agree; no hidden/standard/duplicate selection.
- Same observed history + same AI seed/config gives the same decisions even if hidden opponent queue, unobserved book or combat RNG seed differs. Reaction deadlines are never earlier than the observation latency. Multiple match simulations do not share RNG or mutable observation objects.
- All 14 spell scenarios above are tested, including effect priorities, immunity, reflection of Dispel and timing windows. Death, poison+empty mana, no affordable actions and no viable combo always make progress legally.
- Same input+seed replays exactly; different seeds vary opening/timing over a 1,000-seed corpus. Report distributions; do not assert that every seed must produce a unique action sequence.
- Bot decision p99 budget initially <1 ms on declared test hardware; 100 concurrent bots must keep authoritative loop processing below 100 ms p99. Measure, do not label this achieved until profiled.

Calibration gates: run ≥1,000 simulated matches per tier over all races/legal deck families, report win rate with confidence intervals, duration, idle time, repeated openings, waste/overheal, queue use, reaction histograms and command rejections. Compare to existing pressure/sustain/control as regressions, not as proof of human likeness. Then run blinded human playtests with informed test participants and mixed human/AI recordings: rate tactical coherence, repetition and response timing; collect suspected-AI rate and reasons. No promise of indistinguishability from a small sample. Ship only after eliminating systematic tells in UI/profile/draft and severe skill/build mismatches.

## Source of truth in code

Existing, inspected:
- server: `modules/match/normal_match/{init,matchmaker,join,leave,signal}.go`
- server: `modules/match/ai_match/{init,bot,join}.go`
- server: `modules/match/engine/core/{loop,rejoin,player_setup}.go`
- server: `modules/match/engine/phase/{connecting,lobby,loading,game_countdown,gameover}/phase.go`
- server: `modules/match/engine/phase/game/{ai,phase,reasons,apply_spell_effect}.go`
- server: `modules/match/engine/state/{state,actions,combat,player_state}.go`, `player_state/types.go`
- server: `modules/combat/profile.go`, `modules/progression`, `modules/character/validate.go`, `data/spells/*.yaml`
- server: `vendor/github.com/heroiclabs/nakama-common/runtime/runtime.go`, `cmd/duel-sim`
- client: `Application/ArcaneDuel/Normal/MatchManager.cs`, `Application/ArcaneDuel/Bot/BotMatchManager.cs`

New components and contracts in this note are proposals. Related current contracts: [[matchmaking]], [[server-architecture]], [[combat-v2]], [[spell-system]], [[progression]], [[combat-stat-rules]], [[client-architecture]], [[rpcs]].
