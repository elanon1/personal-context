---
type: project
project: Hexbane
area: plans
status: active
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
---
# Combat restoration implementation plan

> For agentic workers: use superpowers:subagent-driven-development; scope approved by user “przywroc”.

Goal: activate stored stats/races/skills within the current duel_v2 queue and protocol.
Architecture: shared combat.Profile derived from effective stats, skill values and race; match-local random rolls for evasion and skill growth; all damage/healing uses shared formulas through EffectContext; metadata and simulator consume the same profile.
Tech stack: existing Go/Nakama, no new dependencies.
Spec: user approval in this session; retained modules/combat formulas and race traits; current duel_v2 lifecycle remains authoritative for scheduling.

## Rules / decisions
- HP = max(1, round(STR * race health multiplier)); mana=max(1,INT). Existing casting/dodge caps retained.
- Cost=max(0,ceil(base cost * race multiplier)), free remains free. Cast time uses existing DEX/race percentage; recovery/travel unchanged.
- Passive mana=(1+Meditation/50)*(1+DEX*.001)*race multiplier; active meditation adds 10*(1+Meditation/200)*(1+DEX*.001)*race multiplier after existing 800 ms ramp. Passive HP=1/3 per second, blocked while poisoned. Fractional carries retained; no resurrection.
- Damage=(base*(1+Magery/200)+primary stat*.1/pulse count)*school multiplier*(1-total resistance), capped resistance .75. Heal adds INT*.05/pulse count. Current catalog school retained; do not invent new school assignments from lore.
- Dodge once per hostile package at impact, after reflection changes target, not once per periodic pulse. Miss emits spell_impact reason dodged (existing kind). Random stream owned by game phase, deterministic when seeded by simulator/tests.
- Magery/resistance gain rolls on effective damaging impacts/pulses, meditation once per active second; existing chances and +.1, accumulated skill value clamped to100 and persisted by gameover. Bot trackers remain nil.
- Orc paralysis duration applied in scheduler, so queue expiry, snapshot and handler agree. Existing three-second immunity unchanged.
- Keep MatchLog and prior cleanup; do not restore removed buffer or notification scaffolding.
- Keep protocol2/ruleset duel_v2 transport. Update catalog revision for changed effective rules and document required client version rollout. No client writes outside authorized repository.

## Tasks
- [ ] Profile and consumers: combat/profile.go + tests, character/details, core/player_setup, ai_match/bot, cmd/duel-sim and tests. Interface NewProfile(str,int,dex int, meditation float64,r *race.Race) Profile; fields MaxHealth,MaxMana int; CastingTimeBonus,DodgeChance,ManaRegen,HealthRegen,MeditationRegen,ManaCostMultiplier,ParalyzeDurationMultiplier float64; DamageStat,PrimaryElement,SecondaryElement string; PrimaryElementBonus,SecondaryElementBonus float64; SpellResistances map[string]float64.
- [ ] Effects: scale EffectContext.DealDamage/Heal, pulse count helper, handlers avoid duplicate scaling, paralyze scheduler, regression tests. Consume PlayerState.Combat combat.Profile and RecordSkillGain(skills.SkillType).
- [ ] Match integration: PlayerState.Combat, Roll func()float64, SpellManaCost; actions casts/regeneration/gain; skills deterministic-roll helper; game phase random stream, mana checks/events, dodge and regen events; test regressions.
- [ ] Integration: tests first for changed behavior; full Go suite, race suite, vet, seeded simulator. Review final changes. Update server/protocol notes, audit outcome, decisions and journal.

## Source of truth in code
server:modules/combat, modules/skills, modules/race, modules/match, modules/spell_system, modules/character, cmd/duel-sim.
