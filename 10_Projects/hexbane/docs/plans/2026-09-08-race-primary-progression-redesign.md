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

The current14 spells are the **first content pack**, not the complete future catalog. Races were designed for the old spells and need redesign. Every character always has two primary spells: Magic Arrow and Mirror Reflection, plus a progressing set of additional slots, starting at3 and reaching6 (Human7). Magic Arrow primarily breaks mirrors and should deal only1–2 damage. Explore primary upgrades, replacement spells and racial modifications. The user requested a plan, not immediate implementation.

This supersedes the assumption in [[2026-09-08-balance-review]] that the present neutral catalog should define the permanent racial design. Its measurements remain valid for today's pack and code; they are not evidence that future elemental design should be abandoned.

## Recommended architecture

Use three independently understandable layers:

1. **Stats** determine investment in survival, spell power/resources and tempo. Same spendable budget for every race; soft diminishing returns prevent a single-stat extreme from dominating.
2. **Race** supplies a small set of recognizable mechanical affinities/tradeoffs. It does not forbid stat allocations or rely exclusively on content absent from the first pack.
3. **Loadout** has two permanent primary roles plus3–6 drafted/selected spells (Human7). Primary variants are horizontal options; collection and slot entitlement remain separate.

This is a design proposal. Coefficients, new race passives and primary variants below have not been balance-tested or approved for implementation.

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

These are directions, not a list of all passives to apply together. For the first pass, keep primary effects themselves identical across races. Race identity should work without multiplying the combinations of race × primary variant × future spell pack immediately.

Separate mechanical school/damage-type metadata from cosmetic nature. A future pack can introduce new mechanical affinities through explicit contracts. Do not infer them from names or silently repurpose lore fields. Test every new pack with the old one, not only in isolation.

## 3. Primary spells: a permanent tactical foundation

### Magic Arrow — proposed baseline

- Fixed1 damage on an unblocked hit (within the user's1–2 target); no STR, INT, Magery, school or race damage amplification.
- Preserve the interaction ordering: a hostile Arrow consumes the target mirror before the final reflected target's dodge is resolved. It remains a reliable way to spend a mirror charge, not a new way to bypass reflection.
- Shield may absorb it; dodge may prevent final damage; a reflected Arrow deals the same tiny amount. No true-damage or shield-bypass feature is introduced.
- Keep existing cost3, cast.6s and recovery.3s as the initial control values. Revisit cost/timing only after measuring the anti-mirror loop with low damage.
- Do not reward damaging-skill training from repetitive Arrow probing; define training eligibility explicitly instead of making players farm a utility spell.
- Test1 HP lethal, reflection, dodge, absorption and skill100/high-stat/extreme-race cases. It must never become a normal scaling attack again.

### Mirror Reflection — proposed baseline

- Keep one charge, three seconds, existing cost9/cast.5s/recovery.4s as the initial control configuration.
- No progression to multiple charges and no immunity to Arrow.
- Define whether reflected offensive spells use their original offensive potency or the reflector's. Current code uses reflector stats. Recommended prototype: preserve the original spell's offensive potency, but transfer ownership/target for interaction rules; reflection then counters the opponent's spell rather than becoming hidden racial burst. This is a separate effect-context change, not necessary for the Arrow damage fix.

### Development options

1. **Vertical ranks:** +damage, +charges, lower cost without a drawback. Easy progression, but breaks common tactical expectations and adds veteran power. Reject for primary combat power; cosmetic mastery is fine.
2. **Horizontal variants — recommended:** keep the two roles; unlock alternatives selected before a match, one variant per primary. Every variant has a cost and keeps the baseline variant competitive.
3. **Unrestricted replacement with ordinary spells:** introduces loadouts without a mirror counter or without the promised two-primary foundation. Reject initially. A future alternative can replace Arrow/Mirror only if it fulfills the same role and passes the same interaction tests.

Prototype one alternate per role first, not a large upgrade tree:

| Role | Baseline | Alternate hypothesis | Invariant |
|---|---|---|---|
| Arrow | current tempo/cost,1 damage | faster release for higher mana cost, or slower release for lower cost; pick one prototype | always breaks a mirror, damage remains1–2, tradeoff crosses a real100 ms threshold |
| Mirror | current cast/window, one charge | shorter cast with shorter active window, **or** longer window with higher cost; prototype one | one charge, Arrow still consumes it, no stacking |

Variants use only their primary slot and do not consume the3–6/7 optional slots. Selection is fixed for a match. Unlocks add options, not mandatory rank upgrades. Availability should be the same for all races in the first iteration. Racial primary modifications can be tested later as substitutes for another racial trait, never as a free third layer of bonuses. At most one modifier applies; no hidden stacking with primary upgrades.

## 4. Slot and collection progression

The two primaries are separate from additional spell slots at every level. Pack purchases/unlocks increase the owned pool, not the number of slots. An owned spell need not be drafted. Primary variants live in their own selection UI.

User's phrasing starts players at3 additional slots. Current code instead starts Human at4 and others3. Proposed target honoring the shared3-slot start:

| Level | Other races | Human |
|---|---:|---:|
| 1 | 3 | 3 |
| 4 | 4 | 5 |
| 8 | 5 | 6 |
| 12 | 6 | 7 |

Human's racial slot activates with the first slot unlock. This is an explicit proposal replacing the current4-slot Human start; early-level racial budgets must be tested because Human lacks that advantage before4. Do not silently change starting ownership from4 to3 for existing characters: ownership can exceed match slots and stays preserved. For new creation, offer3 starter selections to every race under this proposal.

Do not mix primary variant unlocks with slot unlocks in the first tuning pass. Add variants after the basic3→6/7 curve and two unchanged primary roles work. Exact unlock milestone should be selected against the revised XP pace, not the current8.5-million-XP cap.

## 5. Delivery sequence and validation gates

### Phase A — isolate the primary utility role

Implement fixed Arrow damage and explicit scaling/training eligibility. Keep other primary timings stable. Update spell descriptions, server metadata and client display. Relevant files: `data/spells/magic_arrow.yaml`, `modules/spell_system/spell_effects/events.go`, effect handler tests, `modules/combat/damage.go` if a reusable scaling policy is introduced. Do not hardcode exceptions independently in multiple handlers.

Gate: Arrow damage remains1 across stats/skills/races, mirror interaction remains reliable, no resource or training exploit; compare tactical spell choice and match duration before/after. This phase does not wait for a complete race redesign.

### Phase B — shared stat and spell-power model

Prototype proportional spell-power scaling, shared budget, soft-cap curve, HP curve and DEX breakpoints in isolated simulations. Re-evaluate healing, shield and skill resistance alongside damage. Relevant modules: combat/profile/stats/damage, character validation/allocation/details, progression constants, race model, client stat allocation views.

Gate: legal balanced/specialized builds at levels1/4/8/12/30, skills0/50/100; every earned point remains allocatable; no Arrow scaling regression; competitive time-to-kill and sustainable mana checked independently.

### Phase C — races and migration

Select one working trait per race, establish a common power budget, test each in pack1 and with synthetic future-pack combinations. Update descriptions and add a **new** data migration rather than editing an already-applied seed migration. Deliver reallocation flow before switching saved characters to new caps/budgets. Adjust Human slot ladder and creation consistently if the proposed3-slot start is accepted.

Gate: all15 race pairings with multiple build archetypes, both seats, shared draft samples, alternate AI policies and targeted human playtests. Raw aggregate wins are insufficient. Test early Human separately, and test Human slot value again as new packs arrive.

### Phase D — one variant per primary role

Implement pre-match selection, persistence, ownership/unlock rules and effective metadata for the two alternate prototypes. No mid-match replacement. Keep defaults valid for existing characters; expand the client/server catalog contract deliberately. Profile-derived runtime values must not mutate shared spell definitions.

Gate: all primary-pair interactions and all races; no variant removes the mirror counter, becomes universally better or turns Arrow into an attack. Only then consider racial primary modifiers.

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
