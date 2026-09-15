---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, client, spells, vfx, sfx, icons]
sources: ["client:CLAUDE.md", "client:Resources/Spells/CastPresets/README.md", "client:docs/opcodes/spell-visual-key.md", "client:docs/opcodes/duel-v2-verification.md"]
---

# Spell visuals: which of the 14 server spells have client assets

Current catalog: `magic_arrow, mirror_reflection, firebolt, heavy_bolt, delayed_hex, poison, paralysis, cleanse, mend, greater_heal, regeneration, barrier, dispel, consume_venom`.

| Server id | Icon filename (under Spells/Icons) | Spell VFX | SFX |
|---|---|---|---|
| magic_arrow | magic_arrow.png | MagicArrow.tscn | ElevenLabs, see [[spell-audio]] |
| mirror_reflection | mirror_reflection.png | MirrorWard.tscn | ElevenLabs formation/rupture, see [[spell-audio]] |
| firebolt | firebolt.png | Firebolt.tscn | ElevenLabs, see [[spell-audio]] |
| heavy_bolt | heavy_bolt.png | HeavyBolt.tscn | ElevenLabs, see [[spell-audio]] |
| delayed_hex | delayed_hex.png | DelayedHex.tscn | ElevenLabs, see [[spell-audio]] |
| poison | poison.png | Poison.tscn | ElevenLabs, see [[spell-audio]] |
| paralysis | paralysis.png | Paralysis.tscn | ElevenLabs, see [[spell-audio]] |
| cleanse | cleanse.png | Cleanse.tscn | ElevenLabs, see [[spell-audio]] |
| mend | mend.png | Mend.tscn | ElevenLabs, see [[spell-audio]] |
| greater_heal | greater_heal.png | GreaterHeal.tscn | ElevenLabs, see [[spell-audio]] |
| regeneration | regeneration.png | Regeneration.tscn | ElevenLabs, see [[spell-audio]] |
| barrier | barrier.png | Barrier.tscn | ElevenLabs, see [[spell-audio]] |
| dispel | dispel.png | Dispel.tscn | ElevenLabs, see [[spell-audio]] |
| consume_venom | consume_venom.png | ConsumeVenom.tscn | ElevenLabs, see [[spell-audio]] |

`Resources/Spells/Icons` contains fourteen canonical `<id>.png` files and their Godot import metadata. Generator leftovers, other old icon folders and old spell sounds were removed; Mirror Reflection formation/shatter audio was subsequently restored. `Spell.GetIconPath` retains the aliases needed by current UI.

## Spell icon redesign (2026-09-13)

Twelve generated icons replace the legacy art; Magic Arrow and Mirror Reflection are preserved byte-for-byte. Regeneration now has its own icon. The style and exact prompts for future spells are recorded in [[spell-icon-art-direction]]. Canonical ids take priority over legacy icon aliases in dashboard/lobby/HUD lookups. Source PNGs keep full resolution; Godot import limits runtime spell textures to 512 px. Godot loaded all 14 square textures; the rendered catalog includes 160 px and 48 px previews. Build: zero errors, 11 warnings. Device/in-match visual review remains separate.

## Effect configuration after 2026-09-08 cleanup

All fourteen canonical catalog spells are registered and implemented; `mirror_reflection` resolves to `mirror_ward`. Eleven field scenes share `SpellField` lifecycle/shader with distinct material/geometry profiles; only Magic Arrow and Firebolt currently travel as projectiles. The legacy FX ids in the icon mapping are **icon folder aliases only**. Other legacy `Game/FX` scenes, scripts, particle/texture assets and spell SFX were deleted; Magic Arrow was subsequently implemented as a new procedural shader effect. The current 14-spell gameplay catalog was not removed. See [[spell-effect-system]].

## Cast presentation (gesture, not the projectile)

Independent of the table above, the caster's animation and hand effect are chosen per spell:

1. **`visual_key`** on the `Spell` object (`Core/Spells/Spell.cs:71`), a self-contained 43-char `vfx1_…` string authored in `Dev/ArenaMaps/ArenaMapsDev.tscn` ("Kopiuj klucz"), decoded by `Game/ScenesV3/Components/SpellVisualKey.cs`. Wire format (28 bytes, big-endian binary32 floats): animation id 0–10, effect profile 0–10, flags (tint, trails), ground ring, intensity 0–3, scale 0.5–2, RGBA. Backend storage of this field is **not implemented**; the client only reads it.
2. **Local preset** `Resources/Spells/CastPresets/<spell_id>.tres` (`SpellVisualPreset`: `Animation`, `Effect` 0=auto/1–11, `OverrideColor`, `Tint`, `Intensity`, `EffectScale`, `Trails`, `GroundRing`; directory constant at `SpellVisualPreset.cs:19`). Present today: a preset for every canonical server spell, plus `_template.tres`. `fireball` resolves to the single `firebolt.tres` preset; the former alias file was replaced. The unreferenced `magic_reflection.tres` preset was deleted on 2026-09-15.
3. **Defaults**: `CastAnimationResolver.Resolve` (`spell.animation` from the server if the race has the clip → `attack_1h_01` → `spell_throw`), and the gesture effect assigned to that clip.

Priority is key → preset → default; malformed keys fall back silently. Presets and keys change only the cast look, never timing, damage, projectile or barrier.

Magic Arrow uses an explicit Arcana-blue Impulse cast preset (intensity 0.85, scale 0.8, hidden ground ring), plus the independently registered projectile/impact. Runtime timing, reflection and preview controls: [[spell-effect-system]].

All views resolve spell ids through `SpellEffectConfigurations.Resolve` and instantiate through `SpellEffectFactory`; actor views additionally use `SpellPresentation` for the same anchoring/scaling. Preview scenes must not duplicate scene paths or default effect durations. See [[spell-effect-system]].

## Full catalog field effects (2026-09-08)

| Spell | Form |
|---|---|
| heavy_bolt | Ember furnace strike from above, hot core, pressure front and ejected sparks |
| delayed_hex | Hex fractured seal with shrinking countdown; implosion/rupture on authoritative damage |
| poison | Venom irregular fumes, viscous motes and damage pulse |
| paralysis | Hex three restraint loops and diagonal bindings |
| cleanse | Vitality rising cleansing sweep and outward dust |
| mend | Vitality two converging soft ribbons and a small chest glow |
| greater_heal | Vitality broad chalice, crown and rising light |
| regeneration | Vitality sustained winding ribbon, motes and authoritative healing pulse |
| barrier | Arcana planar hexagonal facets and absorption response; distinct from Mirror Reflection |
| dispel | Arcana counterrotating cutting crescents and scattered rune fragments |
| consume_venom | Venom collapsing droplets and local implosion; no fabricated transfer/healing |

Each new canonical id has its own `.tscn` and cast `.tres`. Shared offline metadata includes verified cast/recovery times, icon mapping and starter flags; live definitions take precedence. VfxTest saves field tuning to the **selected scene**, so a shared script does not spread one spell's tuning to other spells.

Projectiles are permitted when they express a spell well; their absence in these eleven is an artistic/mechanical choice, not a hard constraint. Wrapping geometry is split into front/rear render passes around the animated sprite. See [[spell-effect-system]].

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
- client:Resources/Spells/CastPresets/

Cleanse casting correction (2026-09-08): `cleanse.tres` selects `cast_2h` instead of `area_2h_02`, retaining the vitality Pressure wave profile and palette. This avoids the 3–4-frame wind-up in most race area clips. See [[client-tutorial]] for local tutorial playback fixes.


## Resource organization (2026-09-15)

See [[assets-pipeline]] for the current asset tree and [[2026-09-15-resources-organization]] for the migration. Spell icons now use canonical ids in `Resources/Spells/Icons/<id>.png`; old icon ids and old full spell paths are accepted by `Spell.GetIconPath`. Editable material and race tools live under `.gdignore`d `ArtSource/`. Duplicate character-creation portraits resolve to the identical `Resources/Races/<race>/avatar.png`. The two unreferenced old arena scenes (`GameHud/Main.tscn`, `GameHud/ArcaneDuel/Main.tscn`) and their exclusive resources were deleted. Historical sections above describe earlier states; they do not imply those removed files remain.
