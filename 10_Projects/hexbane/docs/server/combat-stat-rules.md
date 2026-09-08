---
type: project
project: Hexbane
area: server
status: active
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, combat, stats, races, skills]
---

# Combat statistics — duel_v2.4

User-approved shared stats, traits and primary progression implemented 2026-09-08. Tuple: `combat_protocol=2`, `ruleset_id=duel_v2`, `catalog_version=duel_v2.4`. This supersedes the earlier fixed-resource prototype and duel_v2.3 flat-bonus restoration. Implementation is not a live deployment.

## Shared profile

Define `S(x)=x` up to 200, otherwise `200+.5*(x-200)`; negative inputs are floored at zero. Spending has no soft/hard racial cap: only derived combat benefits diminish. `combat.NewProfile` serves setup, bots, character details and simulation. Base/effective stats are equal after migration 000005. Skill coefficients are snapshotted at entry; earned training changes the next match.

| Quantity | Formula / units |
|---|---|
| Max HP | `max(1,round((100+.5*S(STR))*health_multiplier))` |
| Max mana | `max(1,round(S(INT)))` |
| Cast/recovery reduction | `clamp(S(DEX)*.025-race.casting_time_modifier,-15,15)` percentage points |
| Cast and recovery | base seconds × `(1-reduction/100)`; temporary primary tempo affects an eligible cast only |
| Mana cost | `max(0,ceil(baseCost*mana_cost_multiplier))`; free spells stay free |
| Passive mana/sec | `(1+Meditation/50)*(1+S(DEX)*.001)*mana_regen_multiplier` |
| Additional active meditation/sec | `10*(1+Meditation/200)*(1+S(DEX)*.001)*mana_regen_multiplier`, after .8s warm-up |
| Passive HP/sec | `1/3`, blocked by poison, no resurrection |
| Dodge | `min(S(DEX)*perDex,cap)` percent; default .025/8, Shadow .035/15 |
| Paralysis | duration × target trait; Orc .75 |

Race traits: Human +1 slot; Elf regen 1.2; Dark Elf poison/hex damage 1.1; Shadow casting−2 plus dodge; Gnome cost.85; Orc HP 1.05/paralysis.75. All flat stat/school bonuses are zero. See [[progression]]. Passive/active mana share fractional carry, HP carries separately. Poison/paralysis stop meditation; passive mana continues. Travel stays at catalog time. Deadlines use the first 100ms tick on or after the exact deadline, including stat/race fractions.

## Damage, healing and shielding

Every race uses INT for spell power, including Orc. No per-pulse flat stat addition remains.

```
power = 1 + S(INT)/1000
raw = baseDamage * power * (1+Magery/400)
resistance = min(.75, targetSpellResistance/400 + statResistance + perSpellRaceResistance)
damage = round(raw * offenseModifiers * (1-resistance))
heal = round(baseHeal * power)
shield = round(baseShield * power)
```

Ordinary damage has minimum 1. Reflected damage may round to **0**; a low return fraction must not receive repeated minimum-one poison damage. **Magic Arrow always deals exactly 1**, including on return, bypassing all damage/skill/stat/school scaling; shields and package dodge still work. Dark Elf's ×1.1 is applied only to poison and delayed hex. School bonus branches remain for explicit future metadata, but all current race bonuses are zero. Kinetic resistance is `S(STR)*.0005`, mind resistance `S(INT)*.0005`; current neutral attack/defense/support metadata supplies neither type. Cosmetic nature does not infer a damage type.

Dodge rolls once per hostile package, after reflection interception and swap. No repeat dodge on pulses, hex detonation or self-healing. `damage.amount` is HP actually lost, `absorbed` is shield spent; `heal.amount` is actual restoration and `overheal` is excess.

## Primary interception and checkpoints

Full primary graphs, node IDs and resolved config are in [[spell-system]]. A mirror always intercepts one entire hostile package. Below selected primary tier 3 only direct enemy-target damage is returned; statuses/periodic/delayed damage are blocked without return. Tier3 unlocks returning those mechanics, with damage multiplied by the selected fraction. Status durations are not fractionally scaled; the new target's duration rules apply. Original caster offensive stats, Magery and poison/hex multiplier are captured before ownership changes. Apply fraction once, then new-target mitigation once. Reflector stats never replace original offense. Queued poison/hex retains that snapshot; no reflection chains.

The mirror's effect `value` is returned-damage percentage (10–100), while live `remaining` is exactly **1 charge**. Arrow consumes it before the reflected target's dodge roll. Arrow break effects therefore trigger even if the resulting return is dodged.

Arrow tier 3 shortens its own remaining recovery by .1s on a mirror break. Opening/Counterstroke grant a shared non-stacking .1s speedup for the next non-primary cast started within 2s, with a .1s floor. Counterstroke needs an actually applied returned hostile effect; no grant for dodged/fully blocked returns. Arrow Rebate refunds 1 mana at most actual paid, only on mirror break. Mirror Conservation refunds `floor(actual paid/2)` only on unused expiry; consumption, Dispel, replacement and match end do not refund.

## Skill gain and atomic persistence

Human players (all player races; not bots) have gain trackers. The first effective non-Arrow hostile damage in an action (HP or shield) rolls caster Magery and target Spell Resistance once. All effects/pulses share the action's training flag; poison cannot farm a roll per pulse. Dodges, zero damage, utility Arrow and friendly healing grant no damage-training rolls. Meditation rolls once per active second after warm-up and stops when mana fills.

Chances remain 15% below 50, 8% below 75, 3% below 90, .5% below 100; success +.1. Chance includes pending gains. Each skill is capped at 100 and at **+5 per match**; match-local RNG keeps simulation reproducible. Character level 30 does not stop training. Settlement writes gains, level/MP/record changes and the receipt transactionally; retries return the original receipt. Skill milestone MP and study rewards are specified in [[progression]].

## Client contract and verification boundary

Private spell views project primary configuration and effective mana/cast/recovery values onto copies, leaving catalog immutable. Character details expose corresponding milliseconds and derived attributes; public catalog calls expose base values. Snapshot maxima and action deadlines are authoritative. Dodge and passive regeneration use existing impact/heal events; snapshots and action deadlines reflect refunds and tempo. No protocol-number bump is required. Prepared client support accepts catalog 2.4 while retaining local 2.2/2.3 compatibility; local tutorial remains its separate training model.

The earlier restoration's simulation/build report is historical, not validation of this redesign's balance. Current regression tests cover primary paths/checkpoints, damage/reflection, shared stats, training bounds and settlement. Passing unit/build checks does not establish competitive balance or live rollout.

## Source of truth in code

- `server:modules/combat/{profile,stats,damage}.go`
- `server:modules/primary/graph.go`
- `server:modules/match/engine/state/{combat,actions,player_state}.go`
- `server:modules/match/engine/phase/game/apply_spell_effect.go`
- `server:modules/spell_system/spell_effects/{events,engine,queue}.go`
- `server:modules/character/{details,rewards,primary_progression}.go`
- `server:modules/skills/gain.go`
- `client:Core/Characters/StatAllocation.cs`, `client:Core/Match/DuelV2.cs`
