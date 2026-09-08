---
type: project
project: Hexbane
area: plans
status: proposed
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, progression, xp, skills, magic-points, ranked]
---
# XP pacing and ranked access — proposed design

## User requirements

Characters should level rapidly at first, progressively more slowly later, but reach the character-level cap reasonably quickly and enter ranked games. The user explicitly clarified that skills and Magic Points must continue progressing and **must not block ranked access**. Reaching the character level cap is therefore not equivalent to completing every progression system.

Existing code: level30 cap, cumulative XP `100*1.5^(level-2)`, win100/loss30 plus daily bonuses; skills have individual100 caps and gain from combat activity; Magic Points are a separate spell-learning currency currently awarded only on level-ups. Ranked queue/access is a proposed feature here, not an assertion about an existing working ranked system.

## Four separate progression tracks

| Track | Proposed endpoint | What continues afterwards | Ranked eligibility |
|---|---|---|---|
| Character level |30; all earned stat points and optional slot entitlement | no additional stat points beyond cap | proposed level30 requirement |
| Primary development |six graph levels for each primary | redistribution among earned legal paths | no additional requirement to buy/allocate every node |
| Skills |existing per-skill cap100, unless separately redesigned | train unfinished skills in normal and ranked matches | no minimum/max-skill requirement |
| Magic Points |spell-learning currency, separate from skill values | earning/spending after character level30 | no MP balance, collection-completion or purchase requirement |

Do not treat MP as a skill with a100-point cap. The user's clarification establishes that these systems must not gate ranked; it does not establish a numerical MP balance cap or say that completing one skill stops MP income. A future currency/collection cap is a separate economy decision. Do not require ownership of future content packs to keep ranked access.

## Proposed XP curve

Retain30 character levels and+5 stat points per gained level. Replace exponential total requirements with a gently increasing **per-level** cost:

`XP to advance from L-1 to L = 50 + 10*(L-2)`, for2≤L≤30.

Cumulative XP to reach L:

`TotalXP(L) = (L-1)*(5*L+40)` for1≤L≤30.

This gives50 XP for level2,330 XP for the29→30 step, and5510 total XP to reach30. The server currently expects cumulative XP; do not accidentally install the per-step values in `XPForLevel`.

Prototype match rewards:120 XP for a completed win,90 for a completed loss or draw. No daily bonus is assumed in the estimates. The small win premium rewards winning without making a beginner's progress depend heavily on win rate. Existing daily/first-win rewards need an explicit replacement decision; the first tuning run disables them to measure the basic curve. No extra XP for extending a match, dealing more hits or repeatedly casting utility spells.

| Level reached | Total XP | Approximate match equivalent at105 XP/match |
|---|---:|---:|
|2|50|0.5|
|4|180|1.7|
|8|560|5.3|
|12|1100|10.5|
|20|2660|25.3|
|23|3410|32.5|
|30|5510|52.5|

105 XP is the long-run mean at50% wins, not a claim that every game pays105. The first completed game reaches at least level2, potentially3 on a win. Actual threshold crossing depends on the sequence of wins/losses. Extremes without bonuses:46 wins or62 losses reach30. A roughly even record reaches it in about53 games.

At an assumed3–5 minutes per complete queue/draft/match/result cycle,53 games means about2.7–4.4 hours. This is a planning assumption, not measured live retention data. Instrument real cycle time before finalizing the curve; bot simulations alone omit human draft/queue delays. If the desired onboarding time is shorter, scale all XP thresholds together rather than steepening the late curve again.

## Ranked access and primary graph milestones

Proposed server rule: character level30 unlocks ranked regardless of skill values, MP balance, collection completion or whether earned primary points were allocated. Skills remain trainable after entry. No silent normalization to skill100 is introduced by this plan; that would be a separate game rule requiring an explicit decision.

Suggested character-level unlocks for primary graph tiers1–6:1,5,10,16,23,30. Both primaries gain their next tier entitlement at those milestones independently. Thus the checkpoint3 capability arrives around eight games and checkpoint6 entitlement is available by ranked entry. The user has approved six primary tiers and checkpoints, not these character-level milestones; they remain proposals. Human stays4→7 optional slots and others3→6 at existing levels1/4/8/12.

Even though access is not blocked, persisted skill differences may still affect match strength. Measure same-level characters at skill0/50/100 and their actual battle outcomes; reduce excessive skill coefficients or consider skill-aware matching if needed. Do not solve this by adding a hidden minimum skill requirement or forcing collection completion. Rating and character progression are distinct systems.

## Skills after level30

Keep individual skill caps and earn training progress in both normal and ranked. Remove dependence of the opportunity count on poison pulse count or Arrow spam: a bounded contribution per eligible spell/action or per match is the next design task. Reaching level30 neither sets all skills to100 nor stops their growth. Reaching a skill's own cap stops only that skill's gain.

Training pacing should provide continued progression without the current expected roughly31,458 eligible events becoming necessary to be competitive. Exact training rates are separate from the XP curve and are not approved here. Do not change skill caps or introduce a shared aggregate skill cap implicitly.

## Magic Points after level30

Current level-up-only MP income would stop at30, contrary to the requested continuing progression. Add a repeatable source available in both normal and ranked matches, independent of character XP cap and skill caps.

Prototype: one MP per completed rewarded match, with no win/loss difference. This coefficient is illustrative: at current5 MP/spell it means five matches per learned spell and must be tested against future pack prices. Decide whether to retain milestone MP rewards before implementing; **do not blindly stack** repeatable income on today's278 cumulative level-up MP for a catalog costing only60 MP in total. Existing balances and spell ownership must be preserved or migrated explicitly. Never cap a player's skill development because an MP balance is full.

Do not finalize future-pack economy from the first14-spell pack. Access to ranked does not depend on spending MP. The two primary roles remain available without purchasing optional spells.

## Implementation sequence and checks

1. Implement cumulative XP formula and revised match rewards in `modules/progression/{xp,constants}.go`; test levels1/2/30, exact XP boundaries, multi-level gains and zero remaining XP at cap. Verify `get_progression`, `AddExp` and game-over rewards agree.
2. Define conversion for old saved progress before rollout. Raw old XP is not comparable with new thresholds. Preserve earned level and accrued stat/slot entitlements; map fractional progress within an old level to the new interval, with no double-award on the next match. Preserve cap characters at cap.
3. Add a repeatable MP reward transaction to game-over persistence, including at level30. Establish idempotent per-match payouts, and test win/loss/draw and repeated delivery. Define disconnect/forfeit eligibility from actual match rules; do not pay a full reward simply for queueing or abandoning a match.
4. Keep skill payout independent of character XP eligibility and the new MP payout. Test level30 with unfinished skills, partially capped skills and capped skills. Finalize training cadence separately.
5. Implement explicit ranked access validation using character level; test nonmax skills, zero MP, partial spell collection and unallocated primary nodes still qualify. UI must show the same eligibility rule as the server.
6. Add primary tier grants at selected milestones once graph design is approved. Test crossing several milestones in one game and preserving valid allocations. This does not change existing Human slot entitlement.
7. Simulate and instrument time to milestones, not only matches won. Test continuing normal/ranked rewards after level cap and whether skill/collection differences create unacceptable competitive advantages despite unrestricted access.

No gameplay code or database was changed in this planning task. Reward figures, XP curve, primary milestones and level30 ranked gate are proposals; the explicit user requirement is fast-then-slower leveling with skill/MP progression not blocking ranked.

## Source of truth in code

- `server:modules/progression/{xp,constants,magic_points,spell_slots}.go` — present rewards, cumulative XP and entitlements.
- `server:modules/character/{character,db,rpc}.go` — level application, persistence and progression responses.
- `server:modules/match/engine/phase/gameover/phase.go` — match result payouts.
- `server:modules/skills/{gain,types}.go` — current independent caps and event-driven gain.
- `server:modules/match/normal_match/matchmaker.go` — existing unrestricted matchmaking hook; ranked restrictions are not implemented here yet.
- [[2026-09-08-race-primary-progression-redesign]] — six-tier primary graph and user-confirmed Human starting slots.
