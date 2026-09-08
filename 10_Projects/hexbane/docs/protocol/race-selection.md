---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, protocol, races, character-creation]
sources: ["server:docs/client/race-selection.md", "client:docs/client/race-selection.md", "server:docs/client/combat-v2.md", "server:docs/client/client-implementation-prompt.md", "server:RPCs.md"]
---

# Race selection and character creation

The two `race-selection.md` copies are byte-identical; this merges them with the creation section of
`combat-v2.md` and corrects them against code.

## 1. Current race/stat contract (catalog 2.4)

The former race-floor/ceiling and flat-bonus tables were stale after the 2026-09-08 redesign. Current behavior is documented in [[progression]], [[combat-stat-rules]] and [[2026-09-08-redesign-implementation]]. The client mirrors it in `StatAllocation` and the offline `RaceCatalog`.

- `get_races` takes no payload and returns `races`, `success`, `message`. The roster order is Human, Elf, Dark Elf, Shadow, Gnome, Orc; ids match the client art folders.
- Creation spends exactly 400 points across STR/INT/DEX; each has the **shared** floor 10. There are no race-specific stat limits or flat racial grants. Later levels add 5 points per level. Free respec is available outside a match.
- Race traits: Human +1 selectable slot; Elf mana regeneration ×1.2; Dark Elf poison/hex damage ×1.1; Shadow dodge/casting trait; Gnome mana cost ×0.85; Orc health ×1.05 and paralysis duration ×0.75. The detailed combat formulas remain in [[combat-stat-rules]].
- The current creation screen resets the allocation to the shared minima when the selected race changes, then lets the player distribute the remaining points. This session changes presentation, not the allocation rules or RPC payload.

## 2. Mobile race presentation

Cards show the portrait, name and tagline. The detail pane shows lore and active traits. Removed stat-limit bars, minimum STR/INT/DEX lines and starter-bonus sections are not displayed. Selection does not restore desktop padding; a drag scrolls the list without selecting a card. Compact layout hides the duplicate side summary and keeps Back/Next outside the scrolling content.

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

The client validates the name (3–20 characters), shared stat allocation and exact starter selection before calling `create_character`. Success returns `success`, `message`, `character`; the screen displays server validation failures without navigating away. The existing transaction and starter-selection contract remain unchanged by this UI revision.

## After creation

The dashboard opens the character sheet. `get_character_details`, `respec_stats` and `get_primary_progression` supply its data; see [[character-details]]. All four tabs remain reachable in compact mode, with scrollable content and a separate tab strip where needed.

## Source of truth in code
- `server:modules/race/types.go`, `traits.go`, `registry.go` — wire shape, trait keys, in-memory registry.
- `server:db/migrations/000002_reference_data.up.sql` — initial seed; superseded by migration 000005 for racial traits.
- `server:modules/character/validate.go`, `rpc.go`, `db.go`, `character.go` — creation rules, starter count, transaction.
- `server:modules/progression/constants.go` — 400 points, min 10, starting slots 3.
- `server:modules/spell_system/registry.go` (`GetStarter`), `server:data/spells/*.yaml` — starter/standard flags.
- `client:Core/Characters/StatAllocation.cs`, `RaceCatalog.cs`, `RaceTraits.cs` — client maths and fallback roster.
- `client:Game/ScenesV3/CreateCharacter/CreateCharacterScreen.cs` — wizard, pick count.
