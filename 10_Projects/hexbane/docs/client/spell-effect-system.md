---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, client, spells, vfx, architecture]
sources: ["client:Application/ArcaneDuel/README.md"]
---

# Spell effect runtime after legacy cleanup

The client retains only the new **Mirror Reflection** shell (`Game/FX/MirrorWard.cs`, `.tscn`, `.gdshader`). Old projectile/static VFX and their textures, particles, generator output and sound effects were removed on 2026-09-08. Current duel spells and server mechanics remain available, but no projectile or impact VFX is created for them.

Mirror Reflection audio was restored at the user’s request: `WardAudio` plays `formation.wav` at creation and `shatter.wav` on rupture. The formation fades on cancellation/impact; the scene-owned rupture tail survives the shell and frees itself when finished. The SpellFX bus is used with a Master fallback. `VerifyWardAudio.tscn` checks this lifecycle.

Cast presentation is independent and preserved: `RaceSpriteAnimator`, `CastCharge`, `GestureVfx`, `PoseOcclusion`, hand/depth tracks and local `SpellVisualPreset` / `visual_key`. Meditation and arena atmosphere are also preserved.

## Runtime path

- `SpellAnimationController` owns `SpellEffectManager`, resolving player positions and race animators.
- `SpellEffectConfigurations` registers only `mirror_ward`, pointing to `MirrorWard.tscn`, default duration 3 seconds.
- `SpellEffectRegistry` holds the configurations. Config contains id, scene path, duration, parameters and `OnCaster` / `OnTarget` position selection.
- `SpellEffectFactory` caches packed scenes, instantiates an `IStaticEffect` and calls `Play`. An unsupported instance is freed. The inert auto-cleanup timer, render priority, school registry and unused projectile/beam/area branches were removed. The effect owns its lifetime.
- Snapshot `reflection` / `mirror_reflection` effects create one shell per effect instance id, attached to the protected actor and timed to the authoritative end tick. Other statuses stay visible in the HUD without a spell VFX.
- Removal fades the shell; `consumed`, `broken` and `depleted` shatter it. `spell_reflected` also shatters a shell already fading after a removal snapshot. The old straight-line bounce visual was removed.
- `spell_impact` and pending impacts no longer create spell visuals. No duplicate reflection shell is spawned on impact.
- Match end/disposal stops active effects and clears status tracking.

## Interfaces and previews

`ISpellEffect` exposes `IsPlaying`, `Stop`, `GetPosition`. `IStaticEffect` adds `Play(position, duration, parameters)`, `OnStarted`, `OnFinished`. Projectile, area and beam interfaces have no remaining implementation and were removed.

`Game/ScenesV3/VfxTest/VfxTestScreen.tscn` now lists only MirrorWard. `Game/FX/_Previews/MirrorWardPreview.tscn` previews the same shell. `Game/ScenesV3/Dev/ArenaMaps/VerifyMirrorWard.tscn` verifies race attachment, lifetime, directional shatter and live snapshot/event integration.

## Source of truth in code
- client:Core/Spells/ISpellEffect.cs
- client:Core/Spells/SpellEffectRegistry.cs
- client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs
- client:Application/Modules/Spell/Effects/SpellEffectFactory.cs
- client:Application/Modules/Spell/Effects/SpellEffectManager.cs
- client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs
- client:Game/ScenesV3/GameHud/ArcaneDuel/Components/SpellAnimationController.cs
- client:Game/FX/MirrorWard.cs
