---
type: project
project: Hexbane
area: audits
status: complete
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, balance, races, stats, spells, simulation]
---
# Balance review — races, stat caps, skills and spell integration

Review of the current uncommitted `duel_v2.3` server. This is an audit and a proposed tuning direction; **no gameplay parameters or production code were changed**. Contracts read: [[progression]], [[combat-stat-rules]], [[spell-system]], [[combat-v2]].

## Verdict

The restored progression formulas and the redesigned spell catalog do not currently form a balanced system. The largest problems are structural: full flat stat damage on every cheap cast, STR buying both Orc survival and offense, very low HP ceilings for several races, and skill resistance changing the damage/healing economy. Tuning individual spell base values first would hide these problems and need repeating later.

Recommended direction: distinct racial strengths and weaknesses, with meaningful draft choices and every race competitive. This is the working recommendation, not an approved redesign. Keep all restored mechanics; change their coefficients and interactions deliberately.

## Evidence and limitations

The existing `cmd/duel-sim` is a smoke/baseline tool, not a fair race tournament: `(n+seat)%6` pairs only Human–Elf, Elf–Dark Elf, Dark Elf–Shadow, Shadow–Gnome, Gnome–Orc, Orc–Human. It combines level-1 seed stats, zero skills, maximum draft slots and random access to all selectable spells. It does not report participation denominators. Its raw race win totals must not be treated as tier rankings.

For this audit, a temporary harness called the actual production game phase and formulas without modifying them:

- All 15 distinct race pairs; 10 deterministic draft/RNG seeds; all 9 ordered pressure/sustain/control policy combinations; both seats. **2,700 matches per progression scenario**, 900 appearances per race.
- Level 1: legal seed-account stats, skills 0, only starter choices, 3 slots / Human 4.
- Level 12: +55 points, skills 0, full collection available, 6 slots / Human 7.
- Level 30: +145 points, all three skills 100, full collection, 6/7 slots. Higher-level allocations prefer the race's damage stat, then STR, then DEX, respecting effective caps. This is one explicit build family, not an exhaustive optimizer. Skill-100 and level-30 are a combined stress scenario, not evidence isolating either variable.
- Same shuffled spell pool for both opponents within a seed; Human gets the extra pick. No learned-spell availability constraint at levels 12/30. AI controls both sides at its existing 400 ms decision cadence.
- Same-race Magic Arrow-only vs existing AI: 60 matches per race (10 seeds × 3 opponent policies × both seats). Additional Arrow-only vs Firebolt-only and Heavy Bolt-only: 200 per race each (100 seeds × both seats). Both scripted casters wait for the same idle/400 ms opportunities; no extra prequeue advantage. They do not meditate. These short diagnostic duels isolate cast choice, not optimal human strategy.
- Total: **10,860 simulated duels**. Effective stat limits and HP/mana bounds checked during the run. A second full run produced byte-identical result JSON.

Bots follow fixed priorities, sometimes make poor defensive choices and cannot discover an Arrow-first strategy. Draft samples and seeds are reused across pairings. Numbers are diagnostic scores, **not live-player win-rate estimates**, and no statistical independence or confidence interval is claimed. No live database races were queried: reference migrations and legal seed-account spreads are the input. Existing installations might have different stored race rows.

Data and runnable harness snapshots: `2026-09-08-balance-review-data/{results.json,metrics.json,audit.go,simulation.go,fixtures.go}`. Run from the server root:

```sh
go run /absolute/path/to/2026-09-08-balance-review-data/audit.go /absolute/path/to/2026-09-08-balance-review-data/simulation.go /absolute/path/to/2026-09-08-balance-review-data/fixtures.go
```

Create `/private/tmp/hexbane-balance-review` first; output paths in the harness point there. The two copied simulator source files preserve this audit's fixture loader and quiet logger; production engine imports resolve against the server working tree. Source hashes accompany the data so future engine changes are detectable.

## Race tournament

Score = `(wins + draws/2) / appearances`, including timeouts as draws. The table is conditional on the builds/policies above.

| Race | Level 1, skills 0 | Level 12, skills 0 | Level 30, skills 100 |
|---|---:|---:|---:|
| Human | 50.1% | 42.6% | 48.4% |
| Elf | 50.6% | 59.9% | 61.3% |
| Dark Elf | 46.7% | 54.2% | 51.7% |
| Shadow | 39.1% | 27.3% | 22.7% |
| Gnome | 32.0% | 30.6% | 41.6% |
| Orc | 81.6% | 85.4% | 74.3% |

Timeouts: 8/2700 (0.3%), 47/2700 (1.7%), 755/2700 (28.0%) respectively. Orc dominates this test family; Shadow and Gnome underperform. Human’s universal damage bonus is a real advantage but does **not** make it the strongest overall race.

## Current race profiles

Legal level-1 seed-account builds, skills 0. These are examples, not mandatory creation distributions. Meditation column includes passive mana plus active meditation after warm-up.

| Race | Effective STR / INT / DEX | HP / mana | Dodge | Passive mana/s | Active total mana/s | Cast reduction |
|---|---|---|---:|---:|---:|---:|
| Human | 153 / 154 / 153 | 153 / 154 | 3.83% | 1.153 | 12.683 | 3.825% |
| Elf | 120 / 270 / 110 | 120 / 270 | 2.75% | 1.554 | 17.094 | 2.750% |
| Dark Elf | 110 / 280 / 110 | 110 / 280 | 2.75% | 1.110 | 12.210 | 2.750% |
| Shadow | 120 / 150 / 230 | 120 / 150 | 11.50% | 1.230 | 13.530 | 7.750% |
| Gnome | 100 / 210 / 190 | 100 / 210 | 4.75% | 1.190 | 13.090 | 10.750% |
| Orc | 290 / 110 / 110 | 334 / 110 | 2.75% | 1.110 | 12.210 | -0.250% |

### P0 — racial budgets and survivability

- **Orc:** +110 STR, ×1.15 HP, STR as damage stat, and half-duration paralysis. The seed gives 334 HP while simultaneously getting +29 damage per direct cast. A legal level-1 extreme has STR490/INT10/DEX10, 564 HP and +49 flat damage; it has a severe mana constraint, so this is an extremum, not a proven optimal build. A more usable legal STR400/INT100/DEX10 already gives 460 HP and +40 damage. No STR ceiling. Adding STR improves both time-to-die and time-to-kill.
- **Gnome:** seed HP100, hard ceiling110. Cheap mana and faster casts do not compensate consistently for dying before resource efficiency matters. In addition, ceil rounding turns Arrow's `3×.75=2.25` back into cost3: its discount does not apply to the cheapest attack. A mana9 spell costs7 (22.2% reduction), not a full25%.
- **Dark Elf:** HP ceiling120, DEX ceiling140, no signature trait. Its toxic+40%/mind+20% bonuses match no spells. Seed INT280 is only10 higher than Elf270; Elf gets 40% extra mana regeneration. This does not prove strict dominance for every legal build, but the advertised specialization is missing.
- **Shadow:** forced DEX≥200 and ceilings STR150/INT180. Pays for speed that often does not cross a tick threshold and dodge that is probabilistic. At high DEX, 25% chance to negate an entire poison/hex/control package gives high variance, rather than a reliable survival floor.
- **Elf:** regenerative identity works, but water/ice bonuses do not. Large mana pool plus40% regeneration promotes sustain; HP cap160 constrains investment into survival.
- **Human:** +60 total racial points vs100 for most races and110 for Orc, but receives an extra starter/draft slot and **+20% damage on every damaging spell**, because the whole catalog is neutral. Keep the slot identity; an additional universal damage multiplier needs its own explicit budget. Do not infer weakness/strength from point totals alone.

### P0 — school bonuses belong to a different spell design

All 14 spells are `school: neutral`; all per-spell racial resistance maps are empty. Only Human's school bonus is active. STR kinetic and INT mind resistance also never apply. Dark Elf uses the school name `toxic`, while the resistance damage-type enum recognizes `poison`; school identity and damage type must be defined separately if this path is retained.

Do not relabel spells just to activate old race bonuses. There are no water/ice/air/earth attacks in the current mechanical catalog, so assigning a few fire/toxic labels would still strand several races. `nature` is explicitly cosmetic today; making it mechanical requires a contract change.

## Stat caps: actual reachable extremes

There is no infinite stat budget: creation gives400 base points and level30 adds145. Nevertheless, `max=0` permits concentrated builds far beyond the familiar seeds. The following maxima are **independent extremes**, not one build simultaneously attaining all three. The per-stat lower bound is `max(10, racialModifier+1, raceFloor)`; the total point budget and ceilings on other stats may force an even higher actual minimum.

| Race | Max STR / INT / DEX at level 1 | Max STR / INT / DEX at level 30 |
|---|---|---|
| Human | 250 / 250 / 250 | 250 / 250 / 250 |
| Elf | 160 / 459 / 290 | 160 / 604 / 435 |
| Dark Elf | 120 / 480 / 140 | 120 / 625 / 140 |
| Shadow | 150 / 180 / 480 | 150 / 180 / 625 |
| Gnome | 110 / 340 / 340 | 110 / 485 / 485 |
| Orc | 490 / 120 / 130 | 635 / 120 / 130 |

Ceilings also **force** allocations: with all level30 points spent, Orc must have at leastSTR405 because INT≤120 and DEX≤130; Dark Elf must have at leastINT385 because STR≤120 and DEX≤140; Shadow must have at leastDEX315 because STR≤150 and INT≤180. These are restrictions on build choice, not optional specialization. Even at level1 Orc needsSTR≥260 and Dark Elf INT≥240 despite lower declared race floors.

### P1 — DEX and tick breakpoints

DEX supplies .025 percentage points of cast reduction per point, usually .025 points of dodge, and .1% of base mana regeneration. INT gives a mana point, .1 flat damage per spell and .05 healing per heal; Orc STR gives 1.15 HP plus .1 flat damage. Those benefits are not comparable under the present spell costs.

Minimum ideal action spacing is `ceil((personalizedCast+baseRecovery)/.1)*.1`, when the next action is queued. Actual AI spacing can be longer due to400 ms decisions. At the15% cast-speed cap:

- Arrow: `.6×.85+.3=.81` → **.9s**, same as unmodified .9s; its release also stays at the .6s tick.
- Mirror: `.5×.85+.4=.825` → **.9s**, same as unmodified .9s; release stays at .5s.
- Gnome seed Firebolt reaches1.3s vs Human1.4s. Heavy Bolt reaches2.2s vs Human2.4s. Speed is useful on some spells, not universally.
- Orc seed has −.25% cast reduction, so Arrow becomes1.0s and Firebolt1.5s. An extremely small negative bonus can cross a whole tick boundary.

Keep deterministic ticks, but tune DEX against measured release/recovery breakpoints. Do not simply raise the cap or show a smooth percentage as if every point shortens every spell. Consider whether DEX should affect recovery as well; that is a separate tested behavior change.

## Spell integration

### P0 — flat damage erases attack roles

These are actual production formula results for the Human seed INT154, Magery0 against skill resistance0, before target dodge/reflection/shield. Damage is rounded. Time is ideal queued action spacing, not a measured bot cadence.

| Attack | Damage | Mana | Ideal action spacing | Damage/mana |
|---|---:|---:|---:|---:|
| firebolt | 38 | 9 | 1.4s | 4.22 |
| heavy_bolt | 57 | 18 | 2.4s | 3.17 |
| magic_arrow | 23 | 3 | 0.9s | 7.67 |

Arrow went from4 base damage to23 for3 mana. Dark Elf gets32 and Orc33. Full INT/STR damage applies to every cast regardless of its base damage or mana cost. Heavy Bolt pays twice Firebolt's mana and a much longer cast for only57 vs38 Human damage. It also exposes a larger interruption/reflection window. Simply buffing Heavy Bolt's base damage would not repair scaling at all stat values.

Control experiment results below use identical races and stats on both sides. Entries are **Arrow wins / other wins / draws**. The AI column includes defensive decisions; the other columns force one attack repeatedly.

| Race | Arrow vs AI (60) | Arrow vs Firebolt (200) | Arrow vs Heavy Bolt (200) |
|---|---|---|---|
| Human | 48 / 10 / 2 | 28 / 170 / 2 | 28 / 172 / 0 |
| Elf | 54 / 2 / 4 | 12 / 24 / 164 | 200 / 0 / 0 |
| Dark Elf | 40 / 6 / 14 | 12 / 24 / 164 | 186 / 14 / 0 |
| Shadow | 35 / 25 / 0 | 12 / 184 / 4 | 38 / 136 / 26 |
| Gnome | 30 / 28 / 2 | 16 / 184 / 0 | 8 / 192 / 0 |
| Orc | 40 / 16 / 4 | 32 / 162 / 6 | 196 / 4 / 0 |

Thus Arrow is a serious efficiency/role problem, **not a universally dominant duel strategy**. Firebolt often wins the race to lethal; rounding and reaction cadence matter. The current AI undervalues Arrow because it appears last in its attack priorities, so ordinary bot simulations understate the risk of player optimization.

### Spell-by-spell revision order

| Spell | Finding | Proposed treatment |
|---|---|---|
| Magic Arrow | full flat scaling, cost3, universal access, also breaks mirror | Highest priority: scale damage proportionally to base or give explicit spell-power coefficient; preserve cheap probe identity. |
| Firebolt | efficient immediate damage; often fastest practical lethal in tests | Keep as reference attack when comparing damage/time/mana. Recalculate after scaling change. |
| Heavy Bolt | flat bonus paid only once, twice Firebolt's mana, long counter window | Re-establish burst/efficiency reward for commitment after shared scaling fix. |
| Poison | five damage pulses, six-second mana/regen denial;5 skill rolls | Value control separately from raw damage; keep five-pulse total scaling and prevent5× training advantage. |
| Consume Venom | another full stat bonus for a second cast; consumes remaining poison and denial | Measure combo total with actual tick timing, poison ownership, counterplay and two draft slots. Do not compare its isolated damage/mana to Firebolt. |
| Delayed Hex | delayed, cleanseable, no stacking; full bonus once | Price against actual successful detonations, not base damage/action-time alone. |
| Mend | flat INT healing makes repeated small heals efficient | Put healing on an explicit spell-power budget as well; compare at skill0 and100. |
| Greater Heal | longer exposure, less relative flat bonus per action | Preserve meaningful burst healing; retest vs Mend after formula change. |
| Regeneration |5 rounded pulses, full total INT bonus, poison denies pulses | Test overheal, interrupted uptime, poison and dispel. Human seed total35 healing vs25 base. |
| Barrier | fixed22 shield while damage and healing scale | Give explicit defensive scaling or intentionally stable reference power; raw22 can fail against one scaled Arrow. Resistance applies before absorption, so its effective value also grows with resistance. |
| Mirror Reflection | always available, can reverse an entire high-value package | Re-test against cheap probe and long casts after damage fix. Reflection recalculates from the reflector's stats/skills/race, not the original caster's, an important asymmetry for Orc and Human. Decide if intended. |
| Paralysis | cost18 for1s; best value is interrupting paid casts; Orc only.5s | Assess mana denial/interrupt timing, not DPS. Don't buff duration globally to solve Orc. |
| Cleanse | cost6, cancels hex first or poison, restores ability to meditate | Strong utility; keep explicit priority and price using mana denial prevented, especially on Elf. |
| Dispel | strips mirror/barrier/regen; itself reflectable and dodgeable | Situational; Arrow can consume mirror for3 mana, so dispel's broader utility must justify8 and a draft slot. |

## P1 — skills change both balance and progression incentives

At skill100, Magery increases **base damage only** by50%; Spell Resistance cuts the **entire result** by50%. Equal max skills do not cancel: `(1.5×base+flat)×.5 = .75×base+.5×flat`. Healing has no corresponding reduction. Human seed Arrow becomes13 instead of23; Heavy Bolt38 instead of57, while direct healing is unchanged. The level30/skill100 stress test produced755 timeouts out of2700; this demonstrates a combined sustain problem, not a controlled attribution to resistance alone.

Magery and resistance gain on each damaging event, including absorbed damage; poison offers five rolls. Cheap fast Arrow trains much faster per mana than slow expensive hits. Expected qualifying events to reach100 under the current chance tiers are roughly31,458; the final90→100 alone takes20,000 expected events. Meditation progression depends on seconds actually meditating, so better passive regeneration or cost efficiency can reduce opportunities to train. These incentives should reward participation/use without rewarding weak-spell spam or prolonged farm duels.

Proposal: cap one spell's training contribution, decouple chance from periodic pulse count, and use a bounded per-match or activity budget. Test reducing skill damage/resistance coefficients from50% to25% as a candidate; retain the skills and their progression. Separately decide whether competitive matchmaking normalizes power or matches on it.

## P1 — progression and match fairness

- XP is cumulative: level12 requires5,766; level20 requires147,789; level30 requires8,522,269. At100 XP per win, no daily bonuses, the level30 total is about85,223 wins; at a50/50 win/loss split and65 XP/match it is about131,112 matches. Thus145 extra stat points is primarily a theoretical ceiling until the curve is revised.
- All12 non-standard spells cost5 MP. Non-Humans start with3, so the remaining9 cost45 MP; Human's remaining8 cost40. Cumulative MP reaches38 at11 and48 at12, enough for the full collection by12 if spent that way. Collection and draft progression essentially finish while most of the30-level XP curve remains.
- `BeforeMatchmakerAdd` overwrites the query with an empty string; there is no server-enforced level/skill/MMR filter there. Restored power differences can therefore meet directly. A cosmetic race rebalance cannot fix novice/veteran power mismatch.
- Production bots keep level1 stats and zero skills, even when their draft slots are copied from the human. AI-duel difficulty will fall sharply with player progression and varies strongly with bot race.
- No stat respec RPC is registered in `modules/character/init.go`. Cap changes must include a deliberate redistribution/refund path for saved characters exceeding new limits; do not silently clamp and erase points.

## Recommended revision sequence

**A — Recommended: preserve the14-spell neutral design and rebuild race/stat budgets around it.** Keep race identities, all skills, resistance, regen and the current tactical counters. Stop applying unmatched historical school bonuses; replace them with deliberately budgeted traits. Human's extra slot, Elf's mana sustain, Shadow's evasion, Gnome's efficiency and Orc's durability remain recognizable. Dark Elf needs an actual supported specialization rather than an inactive tooltip.

**B — Reintroduce elemental schools.** Requires a coherent school/damage-type schema and attacks representing every racial affinity, followed by collection/draft testing. Larger redesign; merely renaming current fire/poison spells does not complete it.

**C — Normalize PvP power, keep progression elsewhere.** Easiest way to isolate tactical balance, but changes how progression matters in PvP. This is an alternative product direction, not the default recommendation.

For A, run small controlled tuning passes in this order:

1. **Damage/heal scaling.** Candidate to simulate: `baseDamage × (1 + damageStat/1000) × (1 + Magery/400)`, then race and resistance. This preserves proportional attack roles instead of adding the same flat amount to4 and32. At INT154, skills0 and no racial multiplier, Arrow≈5, Firebolt≈18, Heavy≈37. Consider equivalent proportional healing and explicit shield scaling. These are experimental coefficients, not validated replacements.
2. **Survival and race budget.** Candidate HP curve: `100 + .5×STR`, then a small racial multiplier. At current seeds and proposed Orc multiplier1.05, Gnome≈150 HP and Orc≈257, instead of100/334. This compresses extreme survival differences while retaining stat investment. Compare this with raising low STR caps under the existing HP formula; do not apply both blindly.
3. **Caps and traits.** Retest at least balanced, HP-heavy, damage-heavy and DEX-heavy legal builds for every race and level. Replace unbounded primary-stat limits with documented finite targets if build extremes remain dominant. Revisit Gnome/Dark Elf HP ceilings, Orc's combined STR/HP/paralysis package and Shadow's forced DEX. Provisional experiments: Orc HP bonus5% and paralysis reduction25%; Shadow dodge cap15–20%; Human extra slot without neutral+20%. None is a final balance number. Test trait removals individually before combining them.
4. **Spell tuning.** Establish Firebolt as reference; tune Arrow probe/Heavy burst/Poison denial/Hex delayed pressure, then heals and shields. Use successful damage, blocked healing/meditation, forced reactions and draft opportunity cost, not one aggregate DPS score.
5. **Progression and deployment.** Rework the XP curve and skill training budget, decide match power policy, prepare stat redistribution if caps change, and update live race rows through a new migration. Editing the old applied reference migration alone will not update an existing database.

Acceptance targets for the next tuning iteration (design goals, not achieved results): race score about45–55% under multiple build/policy families; inspect pairwise extremes rather than hiding them in averages; less than5% timeout in representative skill bands; a useful role for each selectable spell; an optimized cheap-spell strategy must not invalidate the draft. Duration target needs playtesting; use20–60s as an initial measurement window rather than a forced rule.

## Source of truth in code

- `server:db/migrations/000002_reference_data.up.sql`, `modules/race/{types,traits}.go` — race bonuses, floors/caps, inactive school affinities.
- `server:scripts/seed_dev_accounts.sh`, `modules/character/{validate,character,init}.go` — legal seeds, point budgets, allocation and registered RPCs.
- `server:modules/combat/{profile,stats,damage}.go` — actual scaling, rounding and caps.
- `server:data/spells/*.yaml` — current14-spell catalog.
- `server:modules/spell_system/spell_effects/{events,queue,engine}.go` and `effect_handlers/` — scaling, reflection/status interactions, pulses.
- `server:modules/match/engine/{state/actions.go,phase/game/ai.go,phase/game/phase.go,phase/game/apply_spell_effect.go}` — timing, decision policies, regen, reflection.
- `server:modules/skills/{types,gain}.go`, `modules/progression/{constants,xp,magic_points,spell_slots}.go` — skill and XP economies.
- `server:modules/match/normal_match/matchmaker.go`, `modules/match/ai_match/{bot,join}.go` — power matching and bot setup.
- `server:cmd/duel-sim/{main,fixtures}.go` — original simulator limitations and fixture loading.
