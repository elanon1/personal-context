---
type: project
project: Hexbane
area: plans
status: proposed
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, design, races, stats, primary-spells, progression]
---
# Race, stat-budget and primary-spell redesign — proposed design plan

## Confirmed user direction

The current14 spells are the **first content pack**, not the complete future catalog. Races were designed for the old spells and need redesign. Every character always has two primary spells: Magic Arrow and Mirror Reflection, plus a progressing set of additional slots, starting at3 and reaching6 (Human starts at4 and reaches7). Magic Arrow primarily breaks mirrors and should deal only1–2 damage. The user now selects primary progression: Mirror blocks100% of the intercepted damage at baseline but returns25%, with a development path to100%; another path extends its duration. Arrow should also have two development paths. Human starts with4 additional slots. Branch budgets, intermediate values and unlock pacing remain proposed. The user requested a plan, not immediate implementation.

This supersedes the assumption in [[2026-09-08-balance-review]] that the present neutral catalog should define the permanent racial design. Its measurements remain valid for today's pack and code; they are not evidence that future elemental design should be abandoned.

## Recommended architecture

Use three independently understandable layers:

1. **Stats** determine investment in survival, spell power/resources and tempo. Same spendable budget for every race; soft diminishing returns prevent a single-stat extreme from dominating.
2. **Race** supplies a small set of recognizable mechanical affinities/tradeoffs. It does not forbid stat allocations or rely exclusively on content absent from the first pack.
3. **Loadout** has two permanent primary roles plus3–6 drafted/selected spells (Human7). Primary spells have two development branches each; collection and slot entitlement remain separate.

This is a design proposal. Coefficients, new race passives and primary progression coefficients below have not been balance-tested or approved for implementation.

## 1. Stat budget and caps

### Current behavior

`Character.AllocateStatPoints` computes new effective stats and calls `ValidateEffectiveStatLimits` before mutation. Race ceilings apply after every level-up allocation, not just creation. Race floors are checked too, though positive-only allocation cannot reduce a valid stat below its floor. Restrictive ceilings indirectly force remaining points into another stat. See the audit's level30 examples: Orc STR≥405, Dark Elf INT≥385, Shadow DEX≥315 when every point is spent.

### Options

- **Permanent race-specific hard bounds:** strongest enforced archetypes, but creates allocation traps, invalidates desired builds and makes future content harder to integrate. Not recommended for the redesign.
- **Race bounds only at creation:** more later freedom, but creation remains restrictive and players may have to level out of an unwanted build. Better than current, still not the recommended default.
- **Shared stat budget, recommended race presets and soft diminishing returns:** best fit for expanding content and player-built characters. Recommended.

### Proposed rule

- Preserve a shared base budget:400 at creation and5 per level, up to545 spendable base points at the current level30 cap. Revising the XP curve is a separate task; do not change allocation pace accidentally.
- Replace mandatory racial STR/INT/DEX floors and ceilings with recommended editable starting presets. Maintain basic universal input bounds and enforce the earned budget server-side.
- Proposed initial universal minimum:10 per stat. All three stats remain useful for all races.
- Remove unequal flat racial point grants from the new design; express racial identity through traits. This yields the same total budget, not merely the same base budget with unequal hidden additions. If flat race stat bonuses are retained instead, they need an explicitly equalized budget before implementation; do not combine both models.
- Use a shared soft-cap curve on **combat benefits**, not a race-specific spending prohibition. Prototype effective benefit `x` up to200, then `200 + .5×(x−200)` above200. This is a simulation candidate, not a selected tuning coefficient. Show raw investment and derived effects separately so spending remains legible.
- Do not add a second race-specific total stat cap. Earned points already bound the total. Any future universal per-stat hard ceiling must leave all legally earned points spendable across the other stats.
- Start stat allocation with an immediate preview of HP, mana, damage/healing contribution, dodge and actual cast/recovery tick changes. DEX must buy observable value; investigate tempo/recovery scaling separately from changing the100 ms simulation tick.
- For existing characters, preserve XP, levels, skills, spell ownership and all earned points; grant a full reallocation under the new budget. Explain that old racial stat grants are replaced by traits. Do not silently clamp stored stats.

## 2. Race redesign for multiple content packs

Design one primary mechanic and, only if needed, one small secondary mechanic per race. A race's essential identity must function with the standard spells and pack1; future schools may expand its choices without creating its identity from nothing. Do not promise damage affinities to spells the player cannot obtain.

| Race | Proposed identity | Direction to prototype | Constraint |
|---|---|---|---|
| Human | versatility | extra optional spell slot; broad build freedom | remove universal neutral+20% damage; slot value grows with future collection size |
| Elf | mana management | efficient active meditation/resource recovery | avoid stacking huge mana, passive and active regen multipliers without a shared budget |
| Dark Elf | pressure and disruption | small reward for maintaining or exploiting hostile statuses, using poison/hex/control already in pack1 | one bounded trigger per spell/window; not a bonus per every periodic tick |
| Shadow | tempo and evasion | modest evasion plus a deliberate timing advantage | do not force DEX or make whole-match survival depend on25% package dodge |
| Gnome | spell efficiency | modest cost or cast-efficiency advantage with a visible tradeoff | account for ceil rounding on cheap spells; avoid best mana and cast speed simultaneously |
| Orc | durability and resistance to disruption | moderately higher survival / shorter control | stop making the same STR investment buy full damage and outsized HP; prototype all races using INT for spell power |

These are directions, not a list of all passives to apply together. For the first pass, keep primary effects themselves identical across races. Race identity should work without multiplying the combinations of race × primary allocation × future spell pack immediately.

Separate mechanical school/damage-type metadata from cosmetic nature. A future pack can introduce new mechanical affinities through explicit contracts. Do not infer them from names or silently repurpose lore fields. Test every new pack with the old one, not only in isolation.

## 3. Primary spells: a permanent tactical foundation

### Shared progression proposal

User correction supersedes the earlier recommendation for horizontal-only variants. Primary upgrades may increase power from a weaker baseline. To preserve distinct mature builds, propose a **limited allocation budget per primary**, rather than eventually maximizing both branches. For the first prototype: two spendable points per primary, two ranks per branch; valid mature allocations are2/0,1/1,0/2. Each point advances one rank. This budget and the ranks below are proposals, not user-approved numbers.

Mirror and Arrow have independent point budgets, so developing one does not force abandoning the other. Earn points from explicit progression milestones, not cast spam. Permit redistribution outside a match; freeze both allocations when a match starts. Exact unlock milestones are to be set with the XP redesign. These points neither consume magic points nor alter optional spell slots in this proposal. Keep the default0/0 state valid for existing characters and define any retrospective grants from earned milestones during implementation.

### Mirror Reflection — confirmed direction and proposed ranks

Mirror intercepts one hostile spell package. Baseline protection is already100% for that intercepted package, **not immunity to all attacks during its active window**. The offensive return starts at25%. It remains breakable by Arrow at every rank, has one charge and expires if unused.

| Branch | Rank0 | Rank1 | Rank2 |
|---|---:|---:|---:|
| Returned damage fraction |25%|60%|100%|
| Active duration |3s|4s|5s|

The25%→100% endpoints are user direction. The intermediate60%,3/4/5s duration values and two-point budget are prototype proposals. Keep base cost9, cast.5s and recovery.4s while testing these branches. No extra charges or Arrow immunity.

At a two-point budget, examples are100% return/3s duration,60%/4s, or25%/5s. The first rewards precise counter timing; the last extends the opportunity to catch a spell but also gives the opponent longer to remove it with Arrow. Longer duration is not automatic invulnerability.

Proposed damage contract:

- Snapshot the incoming spell's offensive potency from its original caster, then multiply by the mirror's return fraction. The new target applies its own mitigation once. Do not amplify again using the reflector's INT/STR/Magery/race.
- Example before defender mitigation: an incoming40-damage spell is fully intercepted; rank0 sends10 damage back, rank2 sends40. No30-damage remainder leaks onto the protected player.
- Transfer reflected ownership/target as current interactions require, but retain offensive potency separately. Keep the no-reflection-chain rule.
- Proposed status policy to settle in the design: reflect hostile statuses fully as today; the fraction scales only their damage. Poison pulses and delayed hex detonation retain the fraction captured at reflection, not a newly looked-up rank. Paralysis duration continues to use the final target's rules. This avoids an accidental ambiguous "25% paralysis" mechanic. Status reflection at baseline is therefore stronger than25% of the overall utility of such spells and must be measured separately.
- Arrow's1-damage utility hit remains1 even when reflected; it consumes the mirror regardless of the returned-damage rank. Explicitly test minimum-damage rounding rather than silently relying on it.

### Magic Arrow — proposed baseline and two branches

Fixed1 damage on an unblocked hit, within the user's1–2 target. No STR/INT/Magery/school/racial damage amplification and no damage-growth branch. Shield and dodge still work; mirror consumption remains before the final target's dodge roll. Do not give damaging-skill training for utility Arrow probing.

| Branch | Rank0 | Rank1 | Rank2 |
|---|---:|---:|---:|
| Base cast time |.6s|.5s|.4s|
| Base mana cost |3|2|1|

Recovery remains.3s. All values are prototype proposals. With two points, a player chooses a.4s/cost3 fast probe, a.5s/cost2 hybrid, or a.6s/cost1 economical probe. All deal1 damage. These casting steps deliberately cross100 ms tick boundaries; test actual final timings after stat and race effects. Mana cost never reaches zero.

Primary damage remains independent of race, while any generic cost/cast modifiers must be applied once and shown in effective metadata. Prototype racial primary-specific modifiers only after these branches work; if introduced, they replace part of another racial trait's power budget rather than providing a free extra bonus.

### Replacement scope

Keep exactly the two primary roles and their existing spell identities for this iteration. No unrestricted replacement with ordinary spells, additional primary slots or mid-match redistribution. A later spell replacing a primary must fulfill its tactical role and pass the same counterplay tests.

## 4. Slot and collection progression

User correction is explicit: **Human starts at4**, other races at3. Preserve the existing slot ladder and starting ownership rules:

| Level | Other races | Human |
|---|---:|---:|
|1|3|4|
|4|4|5|
|8|5|6|
|12|6|7|

The two primary spells are carried on top at every level. Primary development does not occupy optional slots. New packs expand the owned spell pool, not the maximum slot count. Human's extra starter and match slot are available from creation; the earlier proposed shared3-slot start is withdrawn.

## 5. Delivery sequence and validation gates

### Phase A — isolate the primary utility role

Implement fixed Arrow damage and explicit scaling/training eligibility. Keep other primary timings stable. Update spell descriptions, server metadata and client display. Relevant files: `data/spells/magic_arrow.yaml`, `modules/spell_system/spell_effects/events.go`, effect handler tests, `modules/combat/damage.go` if a reusable scaling policy is introduced. Do not hardcode exceptions independently in multiple handlers.

Gate: Arrow damage remains1 across stats/skills/races, mirror interaction remains reliable, no resource or training exploit; compare tactical spell choice and match duration before/after. This phase does not wait for a complete race redesign.

### Phase B — shared stat and spell-power model

Prototype proportional spell-power scaling, shared budget, soft-cap curve, HP curve and DEX breakpoints in isolated simulations. Re-evaluate healing, shield and skill resistance alongside damage. Relevant modules: combat/profile/stats/damage, character validation/allocation/details, progression constants, race model, client stat allocation views.

Gate: legal balanced/specialized builds at levels1/4/8/12/30, skills0/50/100; every earned point remains allocatable; no Arrow scaling regression; competitive time-to-kill and sustainable mana checked independently.

### Phase C — races and migration

Select one working trait per race, establish a common power budget, test each in pack1 and with synthetic future-pack combinations. Update descriptions and add a **new** data migration rather than editing an already-applied seed migration. Deliver reallocation flow before switching saved characters to new caps/budgets. Preserve Human's4-slot/4-starter creation and4→7 entitlement.

Gate: all15 race pairings with multiple build archetypes, both seats, shared draft samples, alternate AI policies and targeted human playtests. Raw aggregate wins are insufficient. Test early Human separately, and test Human slot value again as new packs arrive.

### Phase D — primary development branches

Implement the weaker Mirror baseline and two branches for each primary. Deliver stored per-primary point grants/allocations, server validation of earned budgets/rank caps, out-of-match redistribution and immutable match-entry snapshots. Define retroactive point grants for existing characters. Update character progression UI, primary tooltips, effective match metadata, reflection events and supported catalog version together.

The effect context must distinguish original offensive potency from reflected ownership and retain the selected return fraction through poison/hex scheduling. This is a small amount of new content, but affects more than YAML numbers. Relevant areas: character persistence/RPCs, combat damage context, match setup, effect queue/events, client primary selection and combat display.

Gate: baseline25% interception causes zero leaked damage; final100% and hybrid fractions calculate once; poison/hex carry the captured fraction; status policy and Arrow's1-damage exception are explicit; no reflection chain or extra charge; allocations cannot exceed the earned budget or change mid-match. Compare0/0 beginners, mixed ranks and all2/0,1/1,0/2 endpoints across races and spell packs. Only then consider additional racial primary modifiers.

### Phase E — progression and content-pack compatibility

Revise XP/skill training pacing and matchmaking power policy. Extend balance tooling to all race pairs, explicit level/skill/build inputs, participation denominators and optional-content pools. Require every new pack to retain old primary counterplay and old races' basic viability.

Gate: collection growth does not expand optional slot count beyond6/7; no forced purchase of a new pack to obtain the basic mirror counter; monitor match duration, timeout, primary usage, successful counters and spell choices, not only damage totals.

## Source of truth in code

- `server:modules/character/{character,validate,details}.go` — allocation, race bounds, effective stat presentation.
- `server:modules/progression/{constants,spell_slots}.go` — current400/+5 budget and3/4→6/7 slot entitlement.
- `server:modules/combat/{profile,stats,damage}.go` — current flat scaling and race derivation.
- `server:modules/match/engine/phase/game/apply_spell_effect.go` — reflection before dodge, reassigned caster.
- `server:modules/spell_system/spell_effects/events.go` — damage calculation and gain attribution.
- `server:data/spells/{magic_arrow,mirror_reflection}.yaml` — primary defaults.
- `server:db/migrations/000002_reference_data.up.sql` — old race model requiring deliberate replacement.
- `server:modules/match/engine/core/player_setup.go`, `modules/spell_system/standard.go` — standard spells always present.
- `client:Core/Characters/StatAllocation.cs`, `client:Core/Match/DuelV2.cs` — client allocation and protocol integration to inspect before implementation.
