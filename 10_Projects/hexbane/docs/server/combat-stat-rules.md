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
# Restored combat statistics — duel_v2.3

User-approved restoration on 2026-09-08. `combat_protocol=2`, `ruleset_id=duel_v2`, `catalog_version=duel_v2.3`. The queue, draft, recovery, travel, reflection, immunity and 180-second match limit remain. The fixed 200 HP / 100 mana prototype is superseded for server matches.

## Shared profile

`combat.NewProfile` derives the same combat values for player setup, race-based bots, character details and simulation. It consumes **effective** STR/INT/DEX (racial stat modifiers already applied), stored Meditation skill and the race registry entry. Race resistance maps are copied into the match profile. Skill coefficients are fixed at match entry; accumulated gains become stored values at game over and affect subsequent matches.

| Quantity | Formula / units |
|---|---|
| Max HP | `max(1, CalculateMaxHealth(STR, health_multiplier))`; existing helper clamps an omitted/sub-1 health multiplier to 1 |
| Max mana | `max(1, INT)` |
| Cast reduction | `clamp(DEX*0.025 - race.casting_time_modifier, -15, 15)` percentage points |
| Cast duration | base catalog seconds × `(1 - reduction/100)` |
| Mana cost | `max(0, ceil(base mana cost * mana_cost_multiplier))`; free spells remain free |
| Passive mana/sec | `(1 + Meditation/50) * (1 + DEX*0.001) * mana_regen_multiplier` |
| Additional meditation mana/sec | `10 * (1 + Meditation/200) * (1 + DEX*0.001) * mana_regen_multiplier`, after 800 ms warm-up |
| Passive HP/sec | `1/3`; blocked by poison, never resurrects a dead player |
| Dodge chance | `min(DEX * perDex, cap)` percentage points; defaults `perDex=.025`, `cap=8`; Shadow `.05`, `25` |
| Paralysis duration | base duration × target `paralyze_duration_multiplier` (Orc `.5`), before queue deadlines are calculated |

Fractional regeneration carries between ticks and caps against each player's maximum. Poison/paralysis stop meditation; passive mana continues. Meditation gain time stops when mana fills, including larger simulation steps. Recovery and travel use base catalog values and are not shortened by DEX.

Deadlines are reported as the **first 100 ms tick on or after** the exact deadline (`spell_system.DeadlineTick`). Racial/stat scaling can create fractions of a tick; rounding down would tell the client a cast had ended before the server released it. Effect deadlines and equal-time queue priority use the same tick conversion.

## Damage, healing and evasion

All damage handlers share `EffectContext.DealDamage`: direct damage, poison pulses, delayed hex detonation and consume venom. Let `N` be the effect's actual planned number of pulses: positive multiples of interval strictly below duration (`ceil(duration/interval)-1`, minimum 1 for calculation); poison 6s/1s has **5** pulses. The INT/STR bonus is divided across N, so each tick does not gain a whole spell's flat bonus.

```
raw = base * (1 + Magery/200) + damageStat * .1 / N
schoolScaled = raw * (1 + matchingRaceSchoolBonus/100)
resistance = min(.75, targetSpellResistanceSkill/200
                      + statResistance + targetRaceResistanceForSpell)
finalDamage = max(1, round(schoolScaled * (1-resistance)))
healing = round(baseHeal + casterINT*.05/N)
```

`damageStat` is INT, or STR for Orc's `damage_stat=strength`. Primary school match takes precedence over secondary. Resistance values stored per spell are fractions (`.2` = 20%). Stat resistance is STR×.0005 for kinetic and INT×.0005 for mind. Recognized damage metadata is read from spell `type`, then `school`; ordinary `type=attack` is not a damage element.

**Current catalog limitation:** all 14 spells still have `school=neutral` and attack/defense/support types. They therefore receive skill/per-spell resistance but no kinetic/mind stat resistance. Human's neutral school bonus applies; another race's school bonus only applies when catalog metadata matches. No fire/mind/toxic assignments were inferred from names or lore `nature`, whose contract remains cosmetic. The explicit typed branches are tested for future catalog entries.

Dodge rolls **once per hostile package at impact**, after reflection swaps caster/target, and skips all effects in that package. There are no dodge rolls on each poison pulse, on hex detonation or on self-heals. A dodge emits `spell_impact` with `reason=dodged`; reflection still consumes its charge. Damage mitigated by resistance then consumes shields and HP. Lifecycle `damage.amount` is actual HP lost, `absorbed` is shield consumption; `heal.amount` and MatchLog healing are actual restoration, `overheal` records excess.

## Skill gain and persistence

Human player setup creates `skills.NewSkillGains`; bots do not earn/persist gains. Every actual hostile damaging hit/pulse (including shield absorption) rolls caster Magery and target Spell Resistance. Zero damage, dodged packages, status application without damage and friendly healing do not grant those rolls. Meditation rolls once per full second of active meditation after warm-up, not every server tick.

Existing chances are retained: below 50 →15%, below 75 →8%, below 90 →3%, below 100 →0.5%; success adds .1. Each roll uses base value plus already pending gains for its chance/cap; total cannot exceed100. The phase supplies a match-local random stream. Seeded simulation/tests reproduce dodges and gains without package-global random interleaving.

Game over calls `ApplySkillGains`, copies values into the character and persists them through existing `UpdateMatchResult`. Opcode50 exposes previous/current/gained. No schema migration or persistent notification was introduced.

## Client contract

- Private match views copy each catalog spell and expose the player's effective `mana_cost` and `casting_time`; originals remain immutable for other players and damage scheduling. Both selected and standard spells are projected.
- `get_character_details` sets `stat_bonuses_active=true`, returns derived resource/regen attributes and active modifiers/traits, and personalizes spellbook cost/cast time (milliseconds). Public catalog RPCs still return base values.
- Snapshots contain actual `hp_max`/`mana_max`; the HUD must use them instead of 200/100. Cast/recovery fields already carry authoritative deadlines.
- Passive HP recovery uses the existing `heal` event with `reason=passive_regeneration`; dodge uses existing `spell_impact` with `reason=dodged`. No new opcode or event kind.
- Client `DuelVersion.Supported` accepts server2.3 as well as2.2 for the existing local fixed-value tutorial/preview. Tutorial simulation was not converted into the server progression model.
- `MatchLog` remains and retains existing tick-clearing behavior. Removed legacy `CastInterruptions` buffer stays removed; interruption delivery uses the lifecycle sink.

## Source of truth in code

- `server:modules/combat/profile.go`, `stats.go`, `damage.go` — shared profile and formulas
- `server:modules/match/engine/state/combat.go`, `actions.go`, `player_state.go` — cost, spell views, resources, skill gains
- `server:modules/match/engine/core/player_setup.go`, `modules/match/ai_match/bot.go` — production initialization
- `server:modules/match/engine/phase/game/phase.go`, `apply_spell_effect.go`, `ai.go` — random stream, event/deadline integration and evasion
- `server:modules/spell_system/spell_effects/events.go`, `engine.go`, `queue.go` — shared damage/heal, duration scaling, tick conversion
- `server:modules/skills/gain.go`, `modules/match/engine/phase/gameover/phase.go`, `modules/character/db.go` — growth/persistence
- `server:modules/character/details.go`, `modules/spell_system/version.go`, `cmd/duel-sim` — metadata/version/simulation
- `client:Core/Match/DuelV2.cs` — supported versions
