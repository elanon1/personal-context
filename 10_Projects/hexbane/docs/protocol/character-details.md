---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, protocol, character-details, menu]
sources: ["server:docs/API-REFERENCE-v2.md", "server:docs/progression/client/menu-rpc-requirements.md", "client:docs/Server/rpc_get_character_details.md", "client:docs/Server/progression/menu-rpc-requirements.md"]
---

# `get_character_details` — the menu / character-sheet contract

One authenticated call that feeds the Dashboard "More details" screen (Summary, Stats, Spellbook). It
replaces the `get_my_character` + `get_progression` + `get_player_spells` trio; the client falls back to
that trio only when this RPC is unreachable (`client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.cs`).

- Server: `server:modules/character/details.go:156` (`RpcGetCharacterDetails`), response assembled by
  `BuildCharacterDetails` (`details.go:205`).
- Client: `client:Application/Modules/Character/Queries/GetCharacterDetails/` (query, DTOs, handler).

## Request

`{}` or `{"character_id":"char_<user>_<unix>"}`. A non-empty id must equal the caller's own character
(`details.go:182-185`); this endpoint never serves another player's sheet.

## Response

```json
{
  "combat_protocol": 2, "ruleset_id": "duel_v2", "catalog_version": "duel_v2.3",
  "combat_ruleset": "duel_v2",
  "stat_bonuses_active": true,
  "success": true, "message": "",
  "character":   {"id","name","avatar","race_id","race_name","level","wins","losses","win_rate"},
  "progression": {"experience","experience_to_next_level","experience_total_for_level",
                  "unspent_stat_points","magic_points","magic_points_spent","available_magic_points",
                  "spell_slots","next_spell_slot_level"},
  "stats":       {"strength":{"base","racial","effective","min","max"}, "intelligence":{...}, "dexterity":{...}},
  "skills":      {"meditation":{"value","tier","group"}, "spell_resistance":{...}, "magery":{...}},
  "attributes":  {"max_health":200,"max_mana":140,"mana_regen":1.8,"health_regen":0.33},
  "modifiers":   [{"id":"dodge","name":"Dodge chance","value":2.25,"unit":"percent"}, "..."],
  "racial_traits": [{"id":"spell_slot_bonus","name":"Additional spell slots","value":1,"unit":"slots"}],
  "spellbook":   {"spell_slots_used","spells_learned","spell_slots_unlocked","spell_slots_max",
                  "spells":[{"nature","incantation","recovery_time","travel_time","id","name","school",
                             "description","flavor","mana_cost","cast_time","icon_path","is_learned",
                             "is_equipped","magic_points_cost","level_requirement","learned_at_level","standard"}]}
}
```

Errors come back as `{"success":false,"message":"…","modifiers":[],"racial_traits":[]}` with HTTP 200:
`Authentication required`, `Invalid request payload`, `No character found`,
`Character does not belong to the current user`, `Database error`, `Failed to load spellbook`,
`Failed to build response` (`details.go:156-201`).

## Field notes

| Field | Verified behaviour |
|---|---|
| `stat_bonuses_active` | `true`; restored server combat profile is active. Client uses authoritative derived values. |
| `character.win_rate` | `round(wins/(wins+losses)*100)`, `0` with no matches (`details.go:234-237`). |
| `progression.experience_to_next_level` | `progression.XPToNextLevel` — `XPForLevel(level+1) − experience`, `0` at level 30 (`server:modules/progression/xp.go:20-27`). |
| `progression.experience_total_for_level` | `XPForLevel(level+1)` = `100·1.5^(level−1)`; at level 30 the level-30 threshold so the bar renders full (`details.go:255-258`). |
| `progression.spell_slots` | `characters.spell_slots`: 3 at level 1 (+1 for Human), unlocks at levels 4, 8, 12 to a cap of 6 (+1 Human) (`server:modules/progression/constants.go:26-33`, `spell_slots.go:7-21`). |
| `progression.next_spell_slot_level` | Next of `{4,8,12}` above the current level, `0` when maxed (`spell_slots.go:25-32`). |
| `stats.*.racial` | Race modifier alone, so the UI can show `133 + 20 = 153`. |
| `stats.*.min` / `max` | Race floor/ceiling on the **effective** value; `0` = no limit. Enforced by `create_character` and `allocate_stat_points` at every level (`server:modules/character/validate.go:134-162`, `character.go:160-169`). |
| `skills.*.value` | Rounded to 2 decimals. `tier` = `ceil(value/10)`, `0` at 0. `group` = `core` (meditation, magery) or `defense` (spell_resistance). |
| `attributes` | Derived max HP/mana and passive mana/HP per second, rounded for display; formulas in [[combat-stat-rules]]. |
| `modifiers` | `cast_speed`, `dodge`, `spell_power`, `flat_damage`, `healing`, skill/kinetic/mind resistance and `meditation_regen`; typed resistance is conditional on spell metadata. |
| `racial_traits` | Non-neutral slot, cast, school, multiplier, dodge and damage-stat traits plus per-spell resistances, derived from race. |
| `spellbook.spells` | The whole 14-spell catalogue, sorted by school → level requirement → id (`details.go:325-336`), so locked entries render. |
| `spellbook.spells[].cast_time`, `recovery_time`, `travel_time` | Milliseconds. Cost and cast time are personalized from the combat profile; recovery/travel remain base. Public catalog RPCs expose base values. |
| `spellbook.spells[].is_learned` | From `character_spells`; **standard spells are forced `true`** (`details.go:344-346`). `is_equipped` mirrors `is_learned`: there is no loadout outside a match. |
| `spellbook.spells[].magic_points_cost` | `magic_point_cost` from YAML verbatim (`EffectiveMagicPointCost`, `spell.go:36-38`): 5 for every non-standard spell, 0 for standards. The old "5 when unset" fallback no longer exists. |
| `spellbook.spells[].learned_at_level` | Character level when learned; `0` if not learned. |
| `spellbook.spells[].standard` | `true` for `magic_arrow`, `mirror_reflection`. |
| `spellbook.spell_slots_used` / `spells_learned` | Both = number of learned rows (kept for the older client contract). |
| `spellbook.spell_slots_unlocked` | `characters.spell_slots` (draftable today). |
| `spellbook.spell_slots_max` | `MaxSpellSlots (6) + race spell_slot_bonus` (7 for Human). |
| `cooldown` | **Removed**. Not in the struct. |

## Client DTO gaps (`GetCharacterDetailsQuery.cs`)

- `SpellbookDto` maps `spell_slots_used` and `spell_slots_max` only; `spells_learned` and
  `spell_slots_unlocked` are dropped.
- `SpellDetailsDto` declares `starter`, which the server does not send here (always `false`).
- `combat_ruleset` is not mapped (harmless duplicate of `ruleset_id`).

## Related menu RPCs

- Stat allocation: `allocate_stat_points` — request `{"strength","intelligence","dexterity"}` (points to add), response `{"character","message","success"}`; see [[rpcs]].
- Spell purchase: `learn_spell` — `{"spell_id"}` → `{"success","error","spell_id","magic_points_remaining"}`; bounded by MP and level only, never by slots; see [[rpcs]].
- The screen re-queries `get_character_details` after both.
- Level-up rewards arrive only in the game-over message (opcode 50) — see [[level-up-notifications]]; there is no persistent notification.

## Source of truth in code
- `server:modules/character/details.go` — request/response structs, all field derivations.
- `server:modules/progression/constants.go`, `xp.go`, `spell_slots.go` — XP curve, slot unlocks, caps.
- `server:modules/spell_system/spell.go` — millisecond conversions, `EffectiveMagicPointCost`.
- `client:Application/Modules/Character/Queries/GetCharacterDetails/GetCharacterDetailsQuery.cs` — client DTOs.
- `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.cs` — consumer and fallback path.
