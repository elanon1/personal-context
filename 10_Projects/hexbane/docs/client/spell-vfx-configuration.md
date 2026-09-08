---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, client, spells, vfx, sfx, icons]
sources: ["client:CLAUDE.md", "client:Resources/SpellVisuals/README.md", "client:docs/opcodes/spell-visual-key.md", "client:docs/opcodes/duel-v2-verification.md"]
---

# Spell visuals: which of the 14 server spells have client assets

Current catalog: `magic_arrow, mirror_reflection, firebolt, heavy_bolt, delayed_hex, poison, paralysis, cleanse, mend, greater_heal, regeneration, barrier, dispel, consume_venom`.

| Server id | Retained icon folder | Spell VFX | SFX |
|---|---|---|---|
| magic_arrow | magic_arrow | none | none |
| mirror_reflection | mirror_ward | MirrorWard.tscn | formation.wav, shatter.wav |
| firebolt | fireball | none | none |
| heavy_bolt | flamestrike | none | none |
| delayed_hex | explosion | none | none |
| poison | poison_dart | none | none |
| paralysis | paralyze | none | none |
| cleanse | cure | none | none |
| mend | heal | none | none |
| greater_heal | great_heal | none | none |
| regeneration | heal (shared) | none | none |
| barrier | arcane_shield | none | none |
| dispel | gust | none | none |
| consume_venom | venom_shot | none | none |

Each icon folder contains only `<folder>.png` and its Godot import metadata. Generator leftovers, other old icon folders and old spell sounds were removed; Mirror Reflection formation/shatter audio was subsequently restored. `Spell.GetIconPath` retains the aliases needed by current UI.

## Effect configuration after 2026-09-08 cleanup

Only `mirror_ward` is registered and implemented. The legacy FX ids in the icon mapping below are **icon folder aliases only**. All other `Game/FX` scenes, scripts, particle/texture assets and other spell SFX were deleted. The current 14-spell gameplay catalog was not removed. See [[spell-effect-system]].

## Cast presentation (gesture, not the projectile)

Independent of the table above, the caster's animation and hand effect are chosen per spell:

1. **`visual_key`** on the `Spell` object (`Core/Spells/Spell.cs:71`), a self-contained 43-char `vfx1_…` string authored in `Dev/ArenaMaps/ArenaMapsDev.tscn` ("Kopiuj klucz"), decoded by `Game/ScenesV3/Components/SpellVisualKey.cs`. Wire format (28 bytes, big-endian binary32 floats): animation id 0–10, effect profile 0–10, flags (tint, trails), ground ring, intensity 0–3, scale 0.5–2, RGBA. Backend storage of this field is **not implemented**; the client only reads it.
2. **Local preset** `Resources/SpellVisuals/<spell_id>.tres` (`SpellVisualPreset`: `Animation`, `Effect` 0=auto/1–11, `OverrideColor`, `Tint`, `Intensity`, `EffectScale`, `Trails`, `GroundRing`; directory constant at `SpellVisualPreset.cs:19`). Present today: `fireball`, `magic_arrow`, `magic_reflection`, `mirror_reflection`, `_template.tres`. Note `fireball` is keyed by the FX id, not the server id `firebolt`, and `magic_reflection` matches nothing.
3. **Defaults**: `CastAnimationResolver.Resolve` (`spell.animation` from the server if the race has the clip → `attack_1h_01` → `spell_throw`), and the gesture effect assigned to that clip.

Priority is key → preset → default; malformed keys fall back silently. Presets and keys change only the cast look, never timing, damage, projectile or barrier.

## Adding future VFX

New spell VFX need a new implementation and deliberate integration into the duel presentation path. Removed legacy scenes and sound files must not be referenced. Cast presets remain independent; adding a preset does not create a projectile or impact.

## Source of truth in code
- client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs
- client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs
- client:Core/Spells/Spell.cs
- client:Game/ScenesV3/Components/SpellVisualPreset.cs
- client:Game/ScenesV3/Components/SpellVisualKey.cs
- client:Game/FX/MirrorWard.cs
- client:Resources/Spells/
- client:Resources/SpellVisuals/
