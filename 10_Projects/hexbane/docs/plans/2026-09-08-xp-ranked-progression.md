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

Characters should level rapidly at first, progressively more slowly later, but reach the character-level cap reasonably quickly and enter ranked games. The user set the target at approximately80 completed matches to maximum character level. The user explicitly clarified that skills and Magic Points must continue progressing and **must not block ranked access**. Reaching the character level cap is therefore not equivalent to completing every progression system.

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

`XP to advance from L-1 to L = 50 + 17*(L-2)`, for2≤L≤30.

Cumulative XP to reach L:

`TotalXP(L) = 50*(L-1) + 17*(L-1)*(L-2)/2` for1≤L≤30.

This gives50 XP for level2,526 XP for the29→30 step, and8352 total XP to reach30. The server currently expects cumulative XP; do not accidentally install the per-step values in `XPForLevel`.

Prototype match rewards:120 XP for a completed win,90 for a completed loss or draw. No daily bonus is assumed in the estimates. The small win premium rewards winning without making a beginner's progress depend heavily on win rate. Existing daily/first-win rewards need an explicit replacement decision; the first tuning run disables them to measure the basic curve. No extra XP for extending a match, dealing more hits or repeatedly casting utility spells.

| Level reached | Total XP | Approximate match equivalent at105 XP/match |
|---|---:|---:|
|2|50|0.5|
|4|201|1.9|
|8|707|6.7|
|12|1485|14.1|
|20|3857|36.7|
|23|5027|47.9|
|30|8352|79.5|

105 XP is the mean at50% wins, not a fixed payout per game. The first completed game reaches at least level2, potentially3 on a win. Actual threshold crossing depends on result order. Forty wins and forty losses produce8400 XP and reach30. Extremes without bonuses:70 wins or93 losses reach30. The target is approximately80 matches at an even record, not a mandatory80-match eligibility gate. If a fixed number independent of results is desired, that would require different reward rules.

At an assumed3–5 minutes per complete queue/draft/match/result cycle,80 games means approximately4–6.7 hours. This is a planning assumption, not measured live data. Daily XP bonuses and other multipliers are excluded from the80-match estimate; adding them requires retuning. The last level takes526 XP, approximately five average matches, while early slot milestones remain accessible in roughly2/7/14 games.

## Ranked access and primary graph milestones

Proposed server rule: character level30 unlocks ranked regardless of skill values, MP balance, collection completion or whether earned primary points were allocated. Skills remain trainable after entry. No silent normalization to skill100 is introduced by this plan; that would be a separate game rule requiring an explicit decision.

Suggested character-level unlocks for primary graph tiers1–6:1,5,10,16,23,30. Both primaries gain their next tier entitlement at those milestones independently. Thus the checkpoint3 capability arrives around ten games and checkpoint6 entitlement is available by ranked entry. The user has approved six primary tiers and checkpoints, not these character-level milestones; they remain proposals. Human stays4→7 optional slots and others3→6 at existing levels1/4/8/12.

Even though access is not blocked, persisted skill differences may still affect match strength. Measure same-level characters at skill0/50/100 and their actual battle outcomes; reduce excessive skill coefficients or consider skill-aware matching if needed. Do not solve this by adding a hidden minimum skill requirement or forcing collection completion. Rating and character progression are distinct systems.

## Skills after level30

Keep individual skill caps and earn training progress in both normal and ranked. Remove dependence of the opportunity count on poison pulse count or Arrow spam: a bounded contribution per eligible spell/action or per match is the next design task. Reaching level30 neither sets all skills to100 nor stops their growth. Reaching a skill's own cap stops only that skill's gain.

Training pacing should provide continued progression without the current expected roughly31,458 eligible events becoming necessary to be competitive. Exact training rates are separate from the XP curve and are not approved here. Do not change skill caps or introduce a shared aggregate skill cap implicitly.

## Spell ownership must outgrow draft capacity

User correction: handing out one spell with each new slot removes the intended draft dilemma. Aim approximately for6 known spells at4 slots,10 at5 slots and15 at6 slots, with actual ownership depending on MP purchases. These counts exclude the two permanent primary spells. The earlier conversational suggestion of one free spell per slot is withdrawn.

Keep slot entitlements at levels4/8/12, with Human starting4 and finishing7. Supply purchasing power **before** the next slot milestone; do not automatically pick spells for the player or require a collection size to enter ranked. Saved MP is a legitimate choice. A temporarily smaller owned pool does not delete an earned slot; current match setup already clamps usable picks to owned spell count while exposing entitlement.

### Early MP budget — proposed tuning

All current non-standard spells cost5 MP. At that reference price, a non-Human starting with3 spells needs15 cumulative MP for6 owned spells,35 MP for10, and60 MP for15. Proposed rewards replace the current level-up MP schedule rather than stacking on it:

| Reached levels | MP granted per level | Cumulative MP at end of band |
|---|---:|---:|
|2–4|5|15|
|5–8|5|35|
|9–11|5|50|
|12|10|60|
|13–30|2|96|

| Character level | Other races: slots / affordable owned pool | Human: slots / affordable owned pool |
|---|---|---|
|1|3 /3|4 /4|
|4|4 /6|5 /7|
|8|5 /10|6 /11|
|12|6 /15|7 /16|

Affordable pool means all milestone MP spent at5 each, enough catalog content and no extra skill-milestone grants. These are purchasing-power targets, not forced ownership or promises at every price point. Optional more expensive spells produce a smaller collection; do not make expensive mean strictly more powerful. No MP is charged for the slot itself. Primary graph progress uses its own track.

The current pack contains only12 purchasable spells plus2 primaries. The level12 target of15/16 owned optional spells is therefore **not currently possible**, regardless of MP supply. With pack1, the full draft pool tops out at12 for6/7 slots, which still requires leaving cards out. The table is a future expanded-catalog economy target. Surplus MP can be saved; do not count primaries to make the numbers fit or invent content silently.

At the proposed XP pace the first completed game grants at least one level and5 MP, so an affordable extra spell creates a draft choice before the first slot increase. New-character match1 remains the small starter loadout. Reaching level12 takes roughly14 matches in the XP model; learning that much content that quickly requires onboarding/playtesting, not just numerical affordability.

### Continuing MP after character level30

Recommended steady source: convert continued match XP into a separate **spell-study progress bar**. Every500 progress XP grants5 MP, carrying overflow forward. This replaces the earlier flat1-MP-per-match proposal; do not pay both by default.

- Normal and ranked qualifying matches contribute their normal120/90 result XP even after character level caps. There are no additional character levels or stat points from this bar.
- At50% wins the mean is105 progress XP/game: roughly one5-MP spell every4.8 games. This is sustainable even after all skills reach their caps.
- No daily login dependency, cast-count payout or incentive to prolong combat. Rewards derive from completed eligible match results.
- On the match that reaches30, only XP beyond the character-cap threshold starts the study bar. XP already used to reach30 is not counted twice. Award the final level's MP normally; carry leftover study XP and handle multiple reward thresholds atomically.
- Normal and ranked use the same base economy proposal. Practice/private/custom match reward eligibility is a separate explicit rule, not automatic inclusion of every match type.

The study bar is progression UI, not a ranked gate. MP balances, unspent rewards and full skill caps never remove access.

### Connection to skills — bonus milestones, not the sole income

Propose **+5 MP once at25/50/75/100 of each skill**. With the existing three skills this is a maximum of60 MP across that character's skill progression, paid over time. These awards are additional to the ordinary MP curve, so the ownership table above is a baseline, not a hard collection ceiling.

- Awards work before and after character level30, in normal and ranked eligible progression.
- Do not pay MP for every+.1 skill gain: poison pulses, cheap spell spam, race-dependent training opportunities and slower high-skill gains would distort income.
- Reaching100 ends that skill's advancement and milestone awards; the study bar continues supplying MP. No obligation to train unwanted skills for the only available currency source.
- Track each awarded `(character, skill, threshold)` exactly once. A reset, respec, replayed match result or race change must not grant the same milestone again. Detect every crossed threshold, including multiple thresholds in one payout.
- For existing characters, choose and record a one-time retroactive milestone grant during migration. Preserve past ownership/balances; do not repeatedly recompute free awards on login.

This connects learning skills with learning spells without making combat training an endless-money loop. Exact5-MP milestones and500-XP study thresholds are proposals to simulate against skill pacing and future pack prices.

## Implementation sequence and checks

1. Implement cumulative XP formula and revised match rewards in `modules/progression/{xp,constants}.go`; test levels1/2/30, exact XP boundaries, multi-level gains and zero remaining XP at cap. Verify `get_progression`, `AddExp` and game-over rewards agree.
2. Define conversion for old saved progress before rollout. Raw old XP is not comparable with new thresholds. Preserve earned level and accrued stat/slot entitlements; map fractional progress within an old level to the new interval, with no double-award on the next match. Preserve cap characters at cap.
3. Replace the old level-up MP table with the proposed front-loaded table. Add post-cap study XP/reward rollover and one-time skill milestone grants in game-over persistence. Test exact500 thresholds, multiple thresholds, the level29→30 overflow split, level-up/skill grants in the same match, all capped skills and idempotent repeated result delivery. Persist both awarded skill milestones and study remainder transactionally. Define disconnect/forfeit eligibility; queueing or abandoning alone must not pay a full reward.
4. Keep skill payout independent of character XP eligibility and the new MP payout. Test level30 with unfinished skills, partially capped skills and capped skills. Finalize training cadence separately.
5. Implement explicit ranked access validation using character level; test nonmax skills, zero MP, partial spell collection and unallocated primary nodes still qualify. UI must show the same eligibility rule as the server.
6. Add primary tier grants at selected milestones once graph design is approved. Test crossing several milestones in one game and preserving valid allocations. This does not change existing Human slot entitlement.
7. Test MP totals15/35/60/96 at levels4/8/12/30, equal-price affordability for Human and other races, unspent MP, expensive spell purchases and catalog exhaustion at12 optional spells. Simulate and instrument time to milestones, not only matches won. Test continuing normal/ranked rewards after level cap and whether skill/collection differences create unacceptable competitive advantages despite unrestricted access.

No gameplay code or database was changed in this planning task. Reward figures, exact XP formula, primary milestones and level30 ranked gate are proposals; the user-confirmed pacing target is approximately80 matches to maximum character level, with skill/MP progression not blocking ranked.

## Source of truth in code

- `server:modules/progression/{xp,constants,magic_points,spell_slots}.go` — present rewards, cumulative XP and entitlements.
- `server:modules/character/{character,db,rpc}.go` — level application, persistence and progression responses.
- `server:modules/match/engine/phase/gameover/phase.go` — match result payouts.
- `server:modules/skills/{gain,types}.go` — current independent caps and event-driven gain.
- `server:modules/match/normal_match/matchmaker.go` — existing unrestricted matchmaking hook; ranked restrictions are not implemented here yet.
- [[2026-09-08-race-primary-progression-redesign]] — six-tier primary graph and user-confirmed Human starting slots.
