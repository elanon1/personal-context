---
type: project
project: Hexbane
area: audits
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
tags: [hexbane, audit, docs-verification]
---

> Audit report from the 2026-09-07 documentation consolidation (see [[10_Projects/hexbane/_state|_state]] decisions log). Discrepancies here were already applied to the merged notes; the "code smells" and "open questions" sections are the backlog. Paths: `client:` = `~/RiderProjects/hexbane`, `server:` = `~/GolandProjects/hexbane-server`.

# Report: server spell system, progression, database

Agent: `server` (spells + progression + database). Verified against the working trees of both repos on 2026-09-07. All code citations are `path:line` in the server repo unless prefixed `client:`.

## 1. Input → verdict

| Input | Verdict |
|---|---|
| server `docs/spell_system/GUIDE-v2.md` | dropped: stale (3-line pointer stub, no content) |
| server `docs/spell_system/spells.md` | merged into `staging/server/spell-system.md` (catalog, schema, rules); Polish lore section merged into `staging/server/spell-lore.md` |
| server `docs/spell_system/balance-v2.md` | merged into `staging/server/spell-system.md` (balance section) and `staging/plans/2026-09-05-spell-system-redesign.md` (open concerns) |
| server `docs/spell_system/balance-v2.json` | merged into `staging/server/spell-system.md` (headline numbers only) |
| server `docs/spell_system/database-v2.md` | merged into `staging/server/database.md` |
| server `docs/spell_system/spell-lore.md` | merged into `staging/server/spell-lore.md` (translated to English) |
| server `docs/spell_system/verification-v2.md` | open gates merged into `staging/plans/2026-09-05-spell-system-redesign.md`; rest dropped: not documentation (delivery/verification log) |
| server `docs/progression/overview.md` | merged into `staging/server/progression.md` |
| server `docs/progression/race.md` | merged into `staging/server/progression.md` |
| server `docs/progression/stats.md` | merged into `staging/server/progression.md` |
| server `docs/progression/skills.md` | merged into `staging/server/progression.md` |
| server `docs/progression/combat.md` | merged into `staging/server/progression.md` |
| server `docs/progression/match-integration.md` | merged into `staging/server/progression.md` |
| server `docs/progression/modifiers.md` | merged into `staging/server/progression.md` |
| server `docs/progression/progression.md` | merged into `staging/server/progression.md` with corrected XP/MP tables |
| server `docs/superpowers/specs/2026-09-02-race-system-redesign-design.md` | dropped: implemented and then superseded. Roster/limits/traits exist in code and are documented in `progression.md`; every combat trait except `spell_slot_bonus` was disabled by the 2026-09-05 spell redesign, so the spec's fighting-style rationale no longer describes the game |
| server `docs/superpowers/plans/2026-09-02-race-system-redesign.md` | dropped: fully implemented (tasks 1–9 all have code), then tasks 5–8's combat wiring was removed by the spell redesign. Its unchecked verification boxes are moot |
| server `docs/superpowers/specs/2026-09-05-spell-system-redesign.md` | rules merged into `spell-system.md` / `progression.md`; release gates into `staging/plans/2026-09-05-spell-system-redesign.md` |
| server `docs/superpowers/plans/2026-09-05-spell-system-redesign.md` | merged into `staging/plans/2026-09-05-spell-system-redesign.md` (remaining items only) |
| client `docs/Server/progression/overview.md` | dropped: stale (see §2.D) |
| client `docs/Server/progression/race.md` | dropped: stale; roster table re-derived from migration `000002` into `progression.md` |
| client `docs/Server/progression/stats.md` | dropped: stale (all stat combat effects inactive) |
| client `docs/Server/progression/skills.md` | dropped: stale (skill gains/effects inactive) |
| client `docs/Server/progression/combat.md` | dropped: stale (damage formula not on the live path) |
| client `docs/Server/progression/match-integration.md` | dropped: stale |
| client `docs/Server/progression/progression.md` | dropped: stale (slot ladder, XP and MP tables wrong) |
| client `docs/Server/progression/races_seed.sql` | dropped: not documentation and stale (36 legacy races, retired spell ids, missing columns) |
| client `docs/Plans/2026-09-04-standard-spells-6-slot-draft.md` | dropped: fully implemented (deviations listed in §2.E) |

Not in my assignment but seen: server `docs/progression/client/menu-rpc-requirements.md` and client `docs/Server/rpc_get_character_details.md` (rpcs agent).

## 2. Discrepancies found

### A. Server spell docs vs code

| Doc said | Code says |
|---|---|
| `spells.md` title "Spell catalog: duel_v2.1" | `CatalogVersion = "duel_v2.2"` (`modules/spell_system/version.go:6`) |
| `balance-v2.md` / `.json` measure catalog `duel_v2.1` | current catalog is `duel_v2.2`; values unchanged, baseline not re-run (`Makefile:263-264` can reproduce) |
| `spells.md` "Increment the published catalog version when changing balance" | consistent, but the version was bumped for a creation-rule change while the JSON still says v2.1 |
| `spell-lore.md` "palettes contain `primary_color`, `accent_color`" | also `id`, `name`, `root`, `visual_style` (`identity.go:19-26`) |
| `database-v2.md` "Application migration history contains only versions 1 and 2" | three migrations: `000003_local_tutorial` adds `account_tutorials` (`db/migrations/000003_local_tutorial.up.sql`) |
| `database-v2.md` "two seeds producing … 36 learned starters" | superseded by the v2.2 note in the same file: 19 rows (3×5 + 4) from `scripts/seed_dev_accounts.sh:30-31` |
| `overview.md` "`modules/progression/spell_slots.go` defines unlock levels" | `SpellSlotUnlockLevels` is in `constants.go:33`; `spell_slots.go` applies it |
| `stats.md` "minimum of 10 per stat" (base) | validation checks **effective** ≥ 10 and base ≥ 1 (`modules/character/validate.go:48-68`) |
| `progression.md` XP table: L5 338, L10 2 563, L15 19 351, L20 146 116, L25 1 103 068, L30 8 329 228 | `XPForLevel` = `int(100·1.5^(n−2))` (`xp.go:10-18`): L5 337, L10 2 562, L15 19 461, L20 147 789, L25 1 122 274, L30 8 522 269 |
| `progression.md` cumulative MP: 10/30/80/130/205/280 at L5/10/15/20/25/30 | level 1 awards 0 and totals run from level 2 (`magic_points.go:4-34`): 8/28/78/128/203/278 |
| `progression.md` `SpellSlotProgress.MaxSlots // 10` | `MaxSpellSlots + slotBonus` = 6 or 7 (`spell_slots.go:41-47`) |
| `progression.md` match-end snippet `progression.ProcessLevelUp(...)` | live path is `char.AddExp` → `ProcessLevelUpWithSlotBonus` (`character.go:117-135`, `gameover/phase.go:225`) |
| `progression.md` "Standard spells … `mirror_reflection` ('Mirror Reflection')", `magic_arrow` "cheap attack" | correct; both cost mana (9 and 3) |

### B. Race redesign spec/plan (2026-09-02) vs code

| Doc said | Code says |
|---|---|
| Human "Starts with 5 spell slots instead of 4 and caps at 11 instead of 10" | 4 → 7 (`constants.go:26-33`, `spell_slots_test.go:27-38`) |
| Migration `000013_race_roster`, `min_*`/`max_*` "NULL = no limit" | baseline `000001`/`000002`; columns `NOT NULL DEFAULT 0 CHECK ≥ 0`, 0 = no limit (`000001_initial_schema.up.sql:16-25`) |
| Elf mana regen ×1.4, Shadow dodge 0.05/DEX cap 25, Gnome mana cost ×0.75, Orc damage from STR / HP ×1.15 / paralyze ×0.5 | traits are stored and parsed (`race/traits.go`) but have **no consumer**: only `Traits.SpellSlotBonus` is read (`character.go:75,266`, `details.go:210,316`). `paralyze.go` has no multiplier; `TryCastSpellAt` uses `s.ManaCost` directly (`actions.go:26-29`); `BuildPlayerState` sets 200/100 (`player_setup.go:33-34`) |
| "Resulting builds at level 30" HP 110–466, mana 120–385, dodge, cast speed | every character 200 HP / 100 mana, no dodge, fixed cast times |
| Periodic-damage scaling fix (`DamageInstances`) | present in `combat/damage.go:44-75` but `CalculateScaledDamage` has no callers outside tests |
| "Existing characters must be re-normalised" | moot: database reset baseline |

### C. Spell redesign spec (2026-09-05) vs code

| Doc said | Code says |
|---|---|
| "Nowe konto otrzymuje sześć darmowych starterów" | superseded 2026-09-06: choose 3 (Human 4) of 6 (`validate.go:106-129`); plan records the amendment |
| Everything else checked (tick order, mana at cast start, one queue, paralysis-only interrupt, poison/regen 5 pulses, hex due semantics, mirror once, dispel reflectable, consume venom ownership, meditation 0.8 s + 10/s, 180 s draw, seat priority by tick parity) | matches `phase.go`, `queue.go`, `actions.go`, handlers |

### D. OLD client `docs/Server/progression/*`: every value now wrong

**overview.md**
- "Spell slots unlock at levels 4, 6, 10, 15, 20, 25" → 4, 8, 12 (`constants.go:33`).
- Skills table "+50% mana regen / 50% damage reduction / +50% spell damage" → no skill has any combat effect.
- Migrations `000005_create_races`, `000006_extend_characters`, `000007_extend_spells` → `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial`.
- "During Match: apply stat bonuses, track skill triggers" → neither happens (`player_setup.go:77` `SkillGains: nil`).

**race.md**
- "introduced by migration `000013_race_roster`" → `000002_reference_data`.
- "11 spells in the loadout against everyone else's 10", "raises both the starting count (5) and the cap (11)" → 4/7 vs 3/6.
- Level-30 build table (HP 250/160/120/150/110/466; mana 250/370/385/180/300/120; dodge; cast; flat damage bonus) → all inactive; 200/100 for everyone.
- Trait "Consumed by" column: `mana_regen_multiplier` by `core.BuildPlayerState`/`buildDetailsAttributes` → not consumed; `dodge_per_dex`/`dodge_cap` by `combat.CalculateDodgeChance` → function exists, never called; `mana_cost_multiplier` by `state.SpellManaCost` → no such function; `damage_stat` by `CalculateScaledDamage` → never called; `health_multiplier` by `CalculateMaxHealth` → never called; `paralyze_duration_multiplier` by `effect_handlers/paralyze.go` → not there.
- Casting formula `clamp(DEX·0.025 − race.mod, −15, 15)` and "Element Bonuses / Final Damage" → dormant.
- "live server schools are air, earth, fire, ice, lightning, mind, neutral, toxic, water (see `data/spells/`)" → all 14 spells are `school: neutral`; the per-school directories were deleted.
- `Registry.Get(raceId)` "returns nil if not found" → returns `(*Race, bool)` (`registry.go:49-55`).
- Schema block: `name` now `UNIQUE`, all columns `NOT NULL`, limit CHECKs added, `spell_resistances`/`traits` `NOT NULL` (`000001_initial_schema.up.sql:4-26`).
- Shape of `get_races` sample: still correct.

**stats.md**
- Every "Combat Effect": HP = STR, mana = INT, +0.1 dmg/INT, +0.05 heal/INT, 0.05 %/pt resistances, cast 2.5 %/100 DEX cap 15 %, dodge 1.25 %/100 DEX cap 5 %, mana regen 0.05 %/pt → none active.
- Constants block also disagrees with the dormant code: `DodgeChancePerDex` 0.0125 → 0.025, `MaxDodgeChance` 5.0 → 8.0, `ManaRegenPerDex` 0.0005 → 0.001, plus `MinCastingTimeBonus = −15` (`combat/stats.go:11-26`).
- `CalculateMaxHealth(str)` → `(str, healthMultiplier)`; `CalculateDodgeChance(dex)` → `(dex, perDex, cap)`.
- "minimum of 10 each" (base) → effective ≥ 10, base ≥ 1.
- Allocation "Points are validated to not exceed unspent" → true, but partial allocation is allowed and race limits are enforced per stat (`rpc.go:223-256`, `character.go:140-177`).

**skills.md**
- Gain chances 15/8/3/0.5 % and +0.1 exist in `modules/skills` but are never triggered in a duel.
- Effects "+50% mana regen / +50% damage / 50% reduction" → never applied.
- "persisted to the database on match end" → `UpdateMatchResult` writes the columns with unchanged values.
- Match-integration snippets (`TriggerMeditationGain`, `TriggerMageryGain`, `TriggerSpellResistanceGain`) → no callers on the live path (`TakeDamageWithSkillGain*` and `PlayerState.TriggerMageryGain` are unused).

**combat.md**
- Formula `(Base + Magery + IntBonus) × (1 − Resist)` → live damage is the effect value (`damage.go:78-80`); heal is the effect value (`heal.go:73-81`).
- `DamageContext` fields → struct now also has `SpellSchool`, element fields, `CasterStrength`, `CasterDamageStat`, `DamageInstances` (`damage.go:22-49`); still unused.
- "Effect Handler Integration" snippets → handlers do not import `modules/combat`.
- Constants: dodge 0.0125 / cap 5.0 → 0.025 / 8.0 (dormant).

**match-integration.md**
- PlayerState built from stats (`CalculateMaxHealth(effectiveStr)` etc.) → constants 200/100 (`player_setup.go:33-34`).
- Dodge roll `rand.Float64() < DodgeChance/100` → no dodge exists.
- Mana regen `baseRegen × manaRegenMultiplier × (1 + meditationBonus)` → 1/s fixed, +10/s while meditating (`actions.go:56-85`).
- Skill triggers → none.
- `PlayerState` fields `DodgeChance`, `CastingTimeBonus`, `ManaRegenMultiplier`, `RaceSpellResistances`, `RaceCastingTimeModifier` → absent (`player_state.go:12-67`); present instead: `MaxSpellSlots`, `StandardSpells`, `RecoveryUntil`, `MeditationReadyAt`, `ParalyzeImmuneUntil`.
- Win/loss "else { Losses++ }" → losses increment only on `defeat`; draws change neither (`gameover/phase.go:229-236`).
- Level-up via `ProcessLevelUp` → `char.AddExp` with race slot bonus.

**progression.md**
- Slot ladder 4/5/6/7/8/9/10 at 1/4/6/10/15/20/25 → 3/4/5/6 at 1/4/8/12; Human 4/5/6/7.
- `StartingSpellSlots = 4`, `MaxSpellSlots = 10`, `SpellSlotUnlockLevels = {4, 6, 10, 15, 20, 25}` → 3, 6, `{4, 8, 12}` (`constants.go:26-33`).
- "Starter spells available without MP cost" → only the 3 (Human 4) chosen at creation are free; unchosen starters cost 5 MP.
- "Spell slots determine how many spells a character can equip" → draft slots per match only; collection is uncapped (`spellbook/rpc.go:164-169`).
- XP table and cumulative MP table wrong as in §2.A.
- `CalculateSpellSlots(level)` → `(level, bonus)`; `GetSpellSlotProgress(level)` → `(level, slotBonus)`; `MaxSlots // 10` → 6 + bonus.

**races_seed.sql**
- 36 races (`drosskin` … `glassvein`) → 6 (`human, elf, dark_elf, shadow, gnome, orc`).
- Element names `Fire/Arcane/Dark/Holy/Nature` → lowercase `neutral/water/ice/toxic/mind/air/lightning/fire/earth`.
- `spell_resistances` keyed by `flamestrike`, `paralyze`, `explosion`, `fireball`, `poison_dart`, `magic_sparkle` → none of these ids exist; all live rows are `{}`.
- Missing `min_*`/`max_*`/`traits` columns; `ON CONFLICT` upsert form → baseline is a plain `INSERT` in `000002`.

### E. Client plan 2026-09-04 (standard spells + 6-slot draft) vs implementation

| Plan proposed | Implemented as |
|---|---|
| `SpellSlotUnlockLevels = {5, 12, 20}` | `{4, 8, 12}` (`constants.go:33`) |
| `mirror_ward` id, rename to "Mirror Reflection" | id `mirror_reflection`, file `data/spells/mirror_reflection.yaml` |
| `magic_arrow` mana 30 "to be balanced" | 3 |
| `spell_system/standard.go` with `var StandardSpellIDs`, free `IsStandard`/`StandardSpells` | registry methods `GetStandard()`/`IsStandard(id)` driven by the YAML `standard` flag (`standard.go`) |
| migration `0000NN_spell_slots_rebalance` | not needed: database reset, `spell_slots DEFAULT 3 CHECK 3–7` |
| decision "standards keep a mana cost?" | yes: 3 and 9 |
| remove `len(learnedSpells) >= char.SpellSlots` cap in `learn_spell` | removed (`spellbook/rpc.go:164-169`) |
| `PrivatePlayerView.standard_spells`, `spell_slots`, `max_spell_slots` | present (`player_state/types.go:33-39`) |
| bot carries standards | `ai_match/join.go:58-63` |
| lobby filters standards, auto-pick skips them | `lobby/phase.go:130,161,226,275` |
| fix `000013_race_roster` "11 slots" text | migration no longer exists |

## 3. Code smells / dead code noticed

- `get_progression` RPC recomputes XP-to-next with `100·1.5^(level−1)` (off by one level vs `XPForLevel`) and next-slot level from the retired ladder `{4,6,10,15,20,25}` (`modules/character/rpc.go:341-357`). It duplicates `get_character_details.progression` with wrong numbers.
- `set_tutorial_completed` is still registered but always returns an error (`rpc.go:153-155`, `init.go:49-52`).
- `NewCharacter` comment still says "While the roster is empty (race.DefaultRaceId == "")" (`character.go:67-68`).
- `modules/combat` is dormant except `CalculateEffectiveStats`; `modules/skills` gain/trigger helpers, `PlayerState.TakeDamageWithSkillGain`, `TakeDamageWithSkillGainAndEvents`, `TriggerMageryGain`, `ApplySkillGains` are unused on the live path. Game-over still reports `skill_gains` before/after (always equal).
- Seven of eight race trait keys are parsed, validated at startup and served by `get_races` but consumed nowhere.
- `damage.go:24-28,55-58` contain empty `if ctx.Caster != nil { // Combat values are fixed }` blocks; `heal.go:73-81` `calculateHeal` is a no-op wrapper.
- `ReflectionHandler` logs `value` as a percent (`reflectionPercent`) while the catalog value is `1` (`reflection.go:18-21`); cosmetic.
- `spell_system.SpellType` defines only `attack`; YAML uses `defense`/`support` and `type` is never validated (`types.go:3-10`).
- `get_entry_spells` parses `school`/`limit` from the payload and ignores them (`rpc.go:14-38`). `GetSpells` (`db.go:13-22`) ignores its arguments and appears unused.
- `characters` effective-stat defaults 153/154/153 encode Human's +20 (`000001_initial_schema.up.sql:39-41`); harmless but coupled.
- CI has no lint step; `make lint` needs an uninstalled `golangci-lint`.
- `balance-v2.json` is labelled `catalog: duel_v2.1` while code is `duel_v2.2`.
- Server `RPCs.md:198` says playstyles allow "Up to 6 slots (slot_number 1-6)" → code and schema allow 1–7 (`playstyle/validate.go:10-21`, migration). For the rpcs agent.
- Server `docs/progression/client/menu-rpc-requirements.md` header says "Character creation grants six starters automatically" → superseded by the 3/4 choice. For the rpcs agent.
- Client: `Core/Spells/Spell.cs:90` maps `mirror_reflection => mirror_ward`, `firebolt => fireball`, `heavy_bolt => flamestrike` (legacy asset aliases); `Core/Spells/StandardSpells.cs:13` comment cites `data/spells/neutral/**/mirror_ward.yaml`, a path that no longer exists.
- Client `Core/Characters/Character.cs:146` hard-codes `MaxSpellSlots = 6` before the server value arrives; `character.Response` has no `max_spell_slots`, only details/private view do.
- Mixed languages: spec, plan and lore docs in Polish; the rest in English. Staged docs are English per brief.

## 4. Open questions (need a human)

1. Keep or delete the dormant race traits (7 keys), `modules/combat` formulas and `modules/skills` gain logic? They are served by `get_races` and shown by the client race picker, which invites players to expect effects that do not exist.
2. Retire `get_progression` (wrong math, unused by details flow) and the always-failing `set_tutorial_completed` registration, or fix them?
3. Should the balance baseline be re-run and relabelled `duel_v2.2` so the recorded numbers match the shipped catalog version?
4. All spells are `school: neutral`; should the race element columns/bonuses and the `school` field be removed from the contract, or kept for a planned visual split?
5. `stats.md` old rule "minimum 10 per base stat" vs code "effective ≥ 10, base ≥ 1": is the code the intended rule (an Orc with base STR 1 is legal)?
