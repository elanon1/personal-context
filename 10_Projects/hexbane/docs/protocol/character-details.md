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
  "combat_protocol": 2, "ruleset_id": "duel_v2", "catalog_version": "duel_v2.4",
  "combat_ruleset": "duel_v2",
  "stat_bonuses_active": true,
  "success": true, "message": "",
  "character":   {"id","name","avatar","race_id","race_name","level","wins","losses","win_rate"},
  "progression": {"experience","experience_to_next_level","experience_total_for_level",
                  "unspent_stat_points","magic_points","magic_points_spent","available_magic_points",
                  "spell_slots","next_spell_slot_level","study_xp","ranked_eligible","primary_tier"},
  "stats":       {"strength":{"base","racial","effective","min","max"}, "intelligence":{...}, "dexterity":{...}},
  "skills":      {"meditation":{"value","tier","group"}, "spell_resistance":{...}, "magery":{...}},
  "attributes":  {"max_health":200,"max_mana":140,"mana_regen":1.8,"health_regen":0.33},
  "modifiers":   [{"id":"dodge","name":"Dodge chance","value":2.25,"unit":"percent"}],
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
| `progression.experience_total_for_level` | `T(L)=45*(L-1)+6*(L-1)*(L-2)`; next-level cumulative threshold, or 6177 at level 30 (`details.go:255-258`). |
| `progression.spell_slots` | `characters.spell_slots`: 3 at level 1 (+1 for Human), unlocks at levels 7, 11, 16 to a cap of 6 (+1 Human) (`server:modules/progression/constants.go:26-33`, `spell_slots.go:7-21`). |
| `progression.next_spell_slot_level` | Next of `{7,11,16}` above the current level, `0` when maxed (`spell_slots.go:25-32`). |
| `progression.study_xp` | Post-cap study remainder 0–499; each500 XP pays5 MP. |
| `progression.ranked_eligible` | Character level≥30 only; no skills/MP/collection/allocation gate. |
| `progression.primary_tier` | Earned per-primary tier 1–6 at character levels1/5/10/16/23/30. |
| `stats.*.racial` | Zero for all races after migration 000005; base and effective stats match. |
| `stats.*.min` / `max` | Shared minimum 10 and maximum0 (no race ceiling). Earned total budget is enforced server-side; full redistribution uses `respec_stats`. |
| `skills.*.value` | Rounded to 2 decimals. `tier` = `ceil(value/10)`, `0` at 0. `group` = `core` (meditation, magery) or `defense` (spell_resistance). |
| `attributes` | Derived max HP/mana and passive mana/HP per second, rounded for display; formulas in [[combat-stat-rules]]. |
| `modifiers` | `cast_speed`, `dodge`, `spell_power` (Magery percent), `stat_spell_power` (INT percent), `healing` (healing/shield percent), skill/kinetic/mind resistance and `meditation_regen`; typed resistance is conditional on spell metadata. |
| `racial_traits` | Non-neutral slot, cast, school, multiplier, dodge and damage-stat traits plus per-spell resistances, derived from race. |
| `spellbook.spells` | The whole 14-spell catalogue, sorted by school → level requirement → id (`details.go:325-336`), so locked entries render. |
| `spellbook.spells[].cast_time`, `recovery_time`, `travel_time` | Milliseconds. Cost, cast and recovery are personalized from resolved primary config plus combat profile; travel remains base. Public catalog RPCs expose base values. |
| `spellbook.spells[].is_learned` | From `character_spells`; **standard spells are forced `true`** (`details.go:344-346`). `is_equipped` mirrors `is_learned`: there is no loadout outside a match. |
| `spellbook.spells[].magic_points_cost` | `magic_point_cost` from YAML verbatim (`EffectiveMagicPointCost`, `spell.go:36-38`): 5 for every non-standard spell, 0 for standards. The old "5 when unset" fallback no longer exists. |
| `spellbook.spells[].learned_at_level` | Character level when learned; `0` if not learned. |
| `spellbook.spells[].standard` | `true` for `magic_arrow`, `mirror_reflection`. |
| `spellbook.spell_slots_used` / `spells_learned` | Both = number of learned rows (kept for the older client contract). |
| `spellbook.spell_slots_unlocked` | `characters.spell_slots` (draftable today). |
| `spellbook.spell_slots_max` | `MaxSpellSlots (6) + race spell_slot_bonus` (7 for Human). |
| `cooldown` | **Removed**. Not in the struct. |

## Client DTO notes (`GetCharacterDetailsQuery.cs`, 2026-09-08)

- `ProgressionDto` maps `study_xp`, `ranked_eligible`, `primary_tier`; `SpellbookDto` maps `spells_learned` and `spell_slots_unlocked` (used for the spellbook subtitle "draft slots N").
- `SpellDetailsDto` declares `starter`, which the server does not send here (always `false`).
- `combat_ruleset` is not mapped (harmless duplicate of `ruleset_id`).
- Modifier `unit` identifiers are rendered as suffixes by the client (`percent`→`%`, `mana_per_second`→` mana/s`, `multiplier`→`×`, `slots`); zero-valued modifiers are hidden on the summary.

## Related menu RPCs

- Stat allocation: `allocate_stat_points` — request `{"strength","intelligence","dexterity"}` (points to add), response `{"character","message","success"}`; see [[rpcs]].
- Spell purchase: `learn_spell` — `{"spell_id"}` → `{"success","error","spell_id","magic_points_remaining"}`; bounded by MP and level only, never by slots; see [[rpcs]].
- The screen re-queries `get_character_details` after both.
- Level-up rewards arrive only in the game-over message (opcode 50) — see [[level-up-notifications]]; there is no persistent notification.

## Primary path and reallocation UI

The separate `get_primary_progression` response supplies both versioned graphs, selected paths and resolved base configs. `set_primary_path` saves a legal prefix; `respec_stats` redistributes the full400+5×(level−1) budget with min 10. Both reject active-match mutation with code 9. Details refresh after saving. Spell descriptions remain catalog prose; the graph response is authoritative for selected window/return/checkpoint values. See [[rpcs]]. Client side: the **Primary** tab and the inline **Reallocate** mode on Stats — [[duel-v2-client]] "Catalog2.4 progression UI".

## Mobile presentation (2026-09-08)

The four tabs are Summary, Stats, Spellbook and Primary. All compact views can scroll vertically; long labels wrap. The tab strip moves below the header below 1700 logical units. Summary sections/primary cards stack at that width; Stats and Spellbook use one column below 1250 and the summary below 900 or whenever the measured columns exceed the available width. Reflow is reevaluated on resize. Card/tab selection preserves mobile padding. See [[design-system]] and [[2026-09-08-mobile-layout-review]]. No RPC or gameplay changes in this revision.

## Source of truth in code
- `server:modules/character/details.go` — request/response structs, all field derivations.
- `server:modules/progression/constants.go`, `xp.go`, `spell_slots.go` — XP curve, slot unlocks, caps.
- `server:modules/spell_system/spell.go` — millisecond conversions, `EffectiveMagicPointCost`.
- `client:Application/Modules/Character/Queries/GetCharacterDetails/GetCharacterDetailsQuery.cs` — client DTOs.
- `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.cs` — consumer and fallback path.
