---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, races, character-creation]
sources: ["server:docs/client/race-selection.md", "client:docs/client/race-selection.md", "server:docs/client/combat-v2.md", "server:docs/client/client-implementation-prompt.md", "server:RPCs.md"]
---

# Race selection and character creation

The two `race-selection.md` copies are byte-identical; this merges them with the creation section of
`combat-v2.md` and corrects them against code.

## 1. Fetching the roster — `get_races`

`server:modules/race/rpc.go:12`; client `client:Application/Modules/Race/Queries/GetRaces/GetRacesQueryHandler.cs:37`.
No payload. Response `{"races":[<Race>],"success":true,"message":"Races retrieved successfully"}`.
Every field is always present (`server:modules/race/types.go:10-34`):

```json
{"race_id":"orc","name":"Orc",
 "str_modifier":110,"int_modifier":0,"dex_modifier":0,
 "casting_time_modifier":3,"spell_resistances":{},
 "primary_element":"earth","secondary_element":"fire","primary_element_bonus":25,"secondary_element_bonus":12,
 "min_str":250,"max_str":0,"min_int":0,"max_int":120,"min_dex":0,"max_dex":130,
 "traits":{"spell_slot_bonus":0,"mana_regen_multiplier":1,"dodge_per_dex":0,"dodge_cap":0,
           "mana_cost_multiplier":1,"damage_stat":"strength","health_multiplier":1.15,
           "paralyze_duration_multiplier":0.5}}
```

Conventions (`types.go:23-24`, `traits.go:26-52`):
- `0` in `min_*`/`max_*` = no limit.
- `traits.dodge_per_dex` / `dodge_cap` `0` = engine default; other multipliers default to `1`; `damage_stat: ""` = intelligence.
- `race_id` matches the art folder `client:Resources/Races/<race_id>/`.

**In `duel_v2` none of the trait or element values affect combat**: every match runs at 200 HP / 100
mana, `stat_bonuses_active:false` (see [[character-details]]). The only trait with a gameplay effect is
`spell_slot_bonus` (Human +1 draft slot). The client preview reflects this: `StatAllocation.ComputePreview`
returns fixed 200 / 100 / slots (`client:Core/Characters/StatAllocation.cs:305-311`); the old preview
formulas (dodge, cast speed, mana regen per DEX…) are retired.

## 2. The roster (seed `server:db/migrations/000002_reference_data.up.sql:1-39`)

| `race_id` | Name | STR/INT/DEX mod | `casting_time_modifier` | floors (effective) | ceilings (effective) | traits |
|---|---|---|---|---|---|---|
| `human` | Human | +20/+20/+20 | 0 | — | 250/250/250 | `spell_slot_bonus: 1` |
| `elf` | Elf | 0/+70/+30 | 0 | INT 200 | STR 160 | `mana_regen_multiplier: 1.4` |
| `dark_elf` | Dark Elf | 0/+100/0 | 0 | INT 220 | STR 120, DEX 140 | — |
| `shadow` | Shadow | 0/0/+100 | −2 | DEX 200 | STR 150, INT 180 | `dodge_per_dex: 0.05`, `dodge_cap: 25` |
| `gnome` | Gnome | 0/+50/+50 | −6 | INT 150, DEX 150 | STR 110 | `mana_cost_multiplier: 0.75` |
| `orc` | Orc | +110/0/0 | +3 | STR 250 | INT 120, DEX 130 | `damage_stat: strength`, `health_multiplier: 1.15`, `paralyze_duration_multiplier: 0.5` |

Elements: human neutral +20; elf water +25 / ice +12; dark_elf toxic +40 / mind +20; shadow air +25 /
mind +12; gnome lightning +25 / fire +12; orc earth +25 / fire +12. Polish names, taglines and lore live
client-side in `client:Core/Characters/RaceCatalog.cs` (order `human, elf, dark_elf, shadow, gnome, orc`,
line 97), which also serves as the offline fallback roster.

## 3. Stat allocation

Server rules (`server:modules/character/validate.go:29-71`, `progression/constants.go:17-18`):
- `base_strength + base_intelligence + base_dexterity == 400` (`CreationPoints`).
- Each **effective** stat (`base + modifier`, `server:modules/combat/stats.go:107-110`) ≥ 10 (`MinStatValue`).
- Each effective stat inside the race floor/ceiling (0 = none).
- Each base ≥ 1. (The client is stricter: base ≥ 10, `client:Core/Characters/StatAllocation.cs:15`.)

Slider bounds the client derives from the roster (`StatAllocation.Bounds`):
```
lo_i = max(10, min_eff_i - modifier_i)   if min_eff_i > 0 else 10
hi_i = max_eff_i - modifier_i            if max_eff_i > 0 else unbounded (capped by 400 - other floors)
allowed_lo_i = max(lo_i, 400 - sum(hi_j, j != i));  allowed_hi_i = min(hi_i, 400 - sum(lo_j, j != i))
```
The dynamic clamp is mandatory: Shadow's ceilings (STR 150, INT 180 effective) together leave 70 for DEX,
below its 200-effective floor, so STR + INT ≤ 300 base only falls out of the coupled formula.

Legal presets (base → effective): Human 133/134/133 → 153/154/153; Elf 120/200/80 → 120/270/110;
Dark Elf 110/180/110 → 110/280/110; Shadow 120/150/130 → 120/150/230; Gnome 100/160/140 → 100/210/190;
Orc 180/110/110 → 290/110/110. Switching race mid-creation resets to that race's preset.

## 4. Starter spell selection — `get_starter_spells` + `spell_ids`

- `get_starter_spells` (`server:modules/spellbook/rpc.go:375`) returns the six `starter: true` spells
  of catalog `duel_v2.2`: `barrier`, `cleanse`, `firebolt`, `heavy_bolt`, `mend`, `poison`. It is a
  **choice pool**, not an automatic grant (the "six starters granted automatically" banner in the old
  server docs is stale).
- The player picks exactly `StartingSpellCount(race_id)` = **4 for `human`, 3 for every other race**
  (`server:modules/character/validate.go:106-111`). The client mirrors it as
  `BaseSpellSlots + (race == "human" ? 1 : 0)` (`client:Game/ScenesV3/CreateCharacter/CreateCharacterScreen.cs:378`).
- Rejected before saving (`validate.go:113-129`): wrong count, duplicates, unknown id, `standard` spell,
  non-starter spell. Messages: `choose exactly N starter spells`, `duplicate spell_id: x`,
  `spell "x" is not a selectable starter`.
- `magic_arrow` and `mirror_reflection` are standard: never chosen, never bought, always carried in a match.
- The selection costs no MP. Unchosen starters and the six advanced spells cost 5 MP later via `learn_spell`.
- `client:Core/Characters/StatAllocation.cs:20` still defines `StarterSpellCount = 6`; it is the pool size, not the pick count, and nothing uses it for validation.

## 5. Creating the character — `create_character`

`server:modules/character/rpc.go:14`; client `client:Application/Modules/Character/Commands/CreateCharacter/CreateCharacterCommandHandler.cs:45-53`.

```json
{"name":"Grug","avatar":"orc","race_id":"orc",
 "base_strength":180,"base_intelligence":110,"base_dexterity":110,
 "spell_ids":["firebolt","poison","barrier"]}
```

Validation order (`validate.go:83-103`): name 3–20 chars → race exists → stat distribution (section 3)
→ starter spells (section 4). Then: user already has a character → `Character already exists` with the
existing character attached. Character and its `character_spells` rows are written in one transaction
(`server:modules/character/db.go:49-75`), `learned_at_level = 1`. Initial `spell_slots` =
`3 + spell_slot_bonus` (`character.go:105`).

Success: `{"success":true,"message":"Character created successfully","character":{...}}`.
Failure: `{"success":false,"message":"...","character":null}`. Stat messages are safe to show verbatim:
`strength requires at least 250 for race Orc, got 243`, `intelligence may not exceed 120 for race Orc, got 134`.

`avatar` is a free string; the client sends the race id (verified in `CreateCharacterScreen.cs`, unverified
for every path). The stale `CreateCharacterRequest.cs` DTO (`health`, `skills`) is not what the handler sends.

## 6. After creation

`get_character_details.stats.*` carries `base`, `racial`, `effective`, `min`, `max` so the level-up
allocation screen reuses the same clamping. `allocate_stat_points` re-checks the race limits on every
call (`server:modules/character/character.go:160-169`): an Orc never exceeds 120 INT at any level.
`racial_traits[]` only ever contains `spell_slot_bonus` (see [[character-details]]).

## Source of truth in code
- `server:modules/race/types.go`, `traits.go`, `registry.go` — wire shape, trait keys, in-memory registry.
- `server:db/migrations/000002_reference_data.up.sql` — the six seeded races.
- `server:modules/character/validate.go`, `rpc.go`, `db.go`, `character.go` — creation rules, starter count, transaction.
- `server:modules/progression/constants.go` — 400 points, min 10, starting slots 3.
- `server:modules/spell_system/registry.go` (`GetStarter`), `server:data/spells/*.yaml` — starter/standard flags.
- `client:Core/Characters/StatAllocation.cs`, `RaceCatalog.cs`, `RaceTraits.cs` — client maths and fallback roster.
- `client:Game/ScenesV3/CreateCharacter/CreateCharacterScreen.cs` — wizard, pick count.
