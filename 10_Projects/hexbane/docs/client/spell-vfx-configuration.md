---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, spells, vfx, sfx, icons]
sources: ["client:CLAUDE.md", "client:Resources/SpellVisuals/README.md", "client:docs/opcodes/spell-visual-key.md", "client:docs/opcodes/duel-v2-verification.md"]
---

# Spell visuals: which of the 14 server spells have client assets

Server catalog ids ([[combat-v2]], `server:data/spells/*.yaml`): `magic_arrow, mirror_reflection, firebolt, heavy_bolt, delayed_hex, poison, paralysis, cleanse, mend, greater_heal, regeneration, barrier, dispel, consume_venom`.

The client never keys assets by these ids directly. Two mapping tables translate a server id into a **legacy FX id** that names the effect scene, the icon folder and (rarely) the sounds:

- FX: `SpellEffectManager.DuelV2.VisualId` (`Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs:16-20`)
- Icon: `Spell.GetIconPath` (`Core/Spells/Spell.cs:88-96`) → `res://Resources/Spells/<fxid>/<fxid>.png`

| Server id | FX id | Effect scene (`Game/FX/`) | Type | Icon | SFX |
|---|---|---|---|---|---|
| magic_arrow | magic_sparkle | `MagicSparkle.tscn` | Projectile | `magic_arrow/magic_arrow.png` (own) | — |
| mirror_reflection | mirror_ward | `MirrorWard.tscn` | Static | `mirror_ward/mirror_ward.png` | `mirror_reflection/sfx/formation.wav`, `shatter.wav` (`MirrorWard.cs:37-38`, `WardAudio.cs`) |
| firebolt | fireball | `Fireball.tscn` | Projectile | `fireball/fireball.png` | — |
| heavy_bolt | flamestrike | `FlameStrike.tscn` | Static | `flamestrike/flamestrike.png` | — |
| delayed_hex | explosion | `Explosion.tscn` | Static | `explosion/explosion.png` | — |
| poison | poison_dart | `PoisonDart.tscn` | Projectile | `poison_dart/poison_dart.png` | — |
| paralysis | paralyze | `Paralyze.tscn` | Static | `paralyze/paralyze.png` | — |
| cleanse | cure | `Cure.tscn` | Static | `cure/cure.png` | — |
| mend | heal | `Heal.tscn` | Static | `heal/heal.png` | `heal/sfx/cast.mp3`, `impact.mp3` (exported on `Heal.cs:83-85`) |
| greater_heal | great_heal | `GreatHeal.tscn` | Static | `great_heal/great_heal.png` | — |
| regeneration | restore | `Restore.tscn` | Static | **`heal/heal.png` (shared, no dedicated icon)** | — |
| barrier | arcane_shield | `ArcaneShield.tscn` | Static | `arcane_shield/arcane_shield.png` | — |
| dispel | gust | `Gust.tscn` | Projectile | `gust/gust.png` | — |
| consume_venom | venom_shot | `VenomShot.tscn` | Projectile | `venom_shot/venom_shot.png` | — |

Result: all 14 have an effect scene and an icon (regeneration borrows heal's). Only `mend` and `mirror_reflection` have sound. `Resources/Spells/mirror_reflection/` holds only SFX; its icon comes from `mirror_ward/`.

Registered but unmapped legacy FX (dead for the current catalog): `aqua_pulse, ember_burst, frost_cut, spark, stoneguard, reflection` (`SpellEffectConfigurations.cs`), plus `Game/FX/MirrorReflection.tscn` and `Reflection.tscn`. `Resources/Spells/` also contains generator leftovers (`manifest.generated.json`, `spell.json`, `motions/`, `primitives/`, `references/`, `textures/`, `v2/`, a stray `fireball.png`).

## Effect configuration (`SpellEffectConfigurations.cs`)

Each entry is a `SpellEffectConfiguration { SpellId, EffectType (Projectile|Static|AreaOfEffect|Beam), Direction (OnCaster|…), EffectScenePath, Duration, AutoCleanup, Parameters }` (e.g. `CreateHeal`, lines 42-56). `SpellEffectManager` instantiates the scene at the caster/target; effect scenes implement `IProjectileEffect` / `IStaticEffect`. Duel v2 events drive it through `SpellEffectManager.DuelV2.cs` (`spell_impact`, `spell_reflected`, `effect_removed`); a `MirrorWard` shatters on removal reasons `consumed|broken|depleted` (lines 44-56).

## Cast presentation (gesture, not the projectile)

Independent of the table above, the caster's animation and hand effect are chosen per spell:

1. **`visual_key`** on the `Spell` object (`Core/Spells/Spell.cs:71`), a self-contained 43-char `vfx1_…` string authored in `Dev/ArenaMaps/ArenaMapsDev.tscn` ("Kopiuj klucz"), decoded by `Game/ScenesV3/Components/SpellVisualKey.cs`. Wire format (28 bytes, big-endian binary32 floats): animation id 0–10, effect profile 0–10, flags (tint, trails), ground ring, intensity 0–3, scale 0.5–2, RGBA. Backend storage of this field is **not implemented**; the client only reads it.
2. **Local preset** `Resources/SpellVisuals/<spell_id>.tres` (`SpellVisualPreset`: `Animation`, `Effect` 0=auto/1–11, `OverrideColor`, `Tint`, `Intensity`, `EffectScale`, `Trails`, `GroundRing`; directory constant at `SpellVisualPreset.cs:19`). Present today: `fireball`, `magic_arrow`, `magic_reflection`, `mirror_reflection`, `_template.tres`. Note `fireball` is keyed by the FX id, not the server id `firebolt`, and `magic_reflection` matches nothing.
3. **Defaults**: `CastAnimationResolver.Resolve` (`spell.animation` from the server if the race has the clip → `attack_1h_01` → `spell_throw`), and the gesture effect assigned to that clip.

Priority is key → preset → default; malformed keys fall back silently. Presets and keys change only the cast look, never timing, damage, projectile or barrier.

## Adding a spell

1. Add `case "<server_id>" => "<fx_id>"` to both `SpellEffectManager.DuelV2.VisualId` and `Spell.GetIconPath` (or make the FX id equal the server id and skip the mapping).
2. Create `Game/FX/<Name>.tscn` implementing `IProjectileEffect`/`IStaticEffect`, and a `Create<Name>()` entry in `SpellEffectConfigurations.cs`.
3. Put the icon at `Resources/Spells/<fx_id>/<fx_id>.png` (see the `create_spell_icon` skill for style).
4. Optional SFX under `Resources/Spells/<fx_id>/sfx/` loaded by the FX scene (pattern: `MirrorWard.cs`, `Heal.cs`). Optional preset `Resources/SpellVisuals/<server_id>.tres` saved from `ArenaMapsDev`.
5. Verify in `Dev/ArenaMaps/ArenaMapsDev.tscn` ("Rzuć zapisany czar") and `Dev/DuelV2Preview.tscn` (asserts icons for all 14).

## Source of truth in code
- `client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` — server id → FX id, event handling
- `client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs` — FX registry
- `client:Core/Spells/Spell.cs` — icon path mapping, `visual_key` field
- `client:Game/ScenesV3/Components/SpellVisualKey.cs`, `SpellVisualPreset.cs`, `CastAnimationResolver.cs` — cast presentation
- `client:Game/FX/*.tscn|cs` — effect scenes and their sounds
- `client:Resources/Spells/<id>/`, `Resources/SpellVisuals/` — icons, SFX, presets
