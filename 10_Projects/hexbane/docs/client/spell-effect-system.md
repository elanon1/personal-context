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

The client implements all fourteen catalog spells. **Mirror Reflection** (`Game/FX/MirrorWard.*`), **Magic Arrow** (`Game/FX/MagicArrow.*`) and **Firebolt** (`Game/FX/Firebolt.*`) retain their dedicated implementations; the other eleven use their own scenes and shared `SpellField` lifecycle with distinct shader geometry/material profiles. Other old projectile/static VFX and their assets were removed on 2026-09-08. Magic Arrow is a new procedural effect, not a restored legacy asset.

Mirror Reflection audio was restored at the user’s request: `WardAudio` plays `formation.wav` at creation and `shatter.wav` on rupture. The formation fades on cancellation/impact; the scene-owned rupture tail survives the shell and frees itself when finished. The SpellFX bus is used with a Master fallback. `VerifyWardAudio.tscn` checks this lifecycle.

Cast presentation is independent and preserved: `RaceSpriteAnimator`, `CastCharge`, `GestureVfx`, `PoseOcclusion`, hand/depth tracks and local `SpellVisualPreset` / `visual_key`. Meditation and arena atmosphere are also preserved.

## Runtime path

- `SpellAnimationController` owns `SpellEffectManager`, resolving player positions and race animators.
- `SpellEffectConfigurations` registers `mirror_ward` (`MirrorWard.tscn`, 3 seconds) `magic_arrow` (`MagicArrow.tscn`, cosmetic flight 0.14 seconds), and `firebolt` (`Firebolt.tscn`, cosmetic flight 0.18 seconds).
- `SpellEffectRegistry` holds the configurations. Config contains id, scene path, duration, parameters and `OnCaster` / `OnTarget` position selection.
- `SpellEffectFactory` caches packed scenes, instantiates an `IStaticEffect` or `IProjectileEffect` and calls the corresponding `Play` overload. An unsupported instance is freed. The effect owns cleanup; no generic timer or beam/area branches are used.
- Snapshot `reflection` / `mirror_reflection` effects create one shell per effect instance id, attached to the protected actor and timed to the authoritative end tick. Other statuses stay visible in the HUD without a spell VFX.
- Removal fades the shell; `consumed`, `broken` and `depleted` shatter it. `spell_reflected` also shatters a shell already fading after a removal snapshot. The old straight-line bounce visual was removed.
- Magic Arrow and Firebolt release, reflection, impact and pending snapshots are handled by `SpellEffectManager.Projectiles.cs`; see the timing rules below. No duplicate reflection shell is spawned on impact.
- Match end/disposal stops active effects and clears status tracking.

## One playback path for every view

`SpellEffectConfigurations.Resolve(spellId)` is the VFX catalog entry point: it normalizes mirror aliases, honours registered overrides, and returns a copy so a preview cannot accidentally mutate the catalog. Scene paths and default effect durations live only in `GetDefaultConfigurations`.

Actor-based views (live match manager, `ReferenceHud.AnimateCast` used by ArenaMapsDev, the shared ProjectilePreview driver, and `RaceSpriteAnimator.ShowReflection`) all call `Game/FX/SpellPresentation.Play`. It resolves the catalog, delegates instantiation/play to `SpellEffectFactory`, places/scales the projectile from palm to target bounds and parents protection shells to their protected actor. This ownership also makes ArenaMapsDev’s barrier-impact controls work consistently.

Point-based tools (`VfxTestScreen` and generic `FxPreview`) use the same resolver/factory; they supply positions instead of actors. VfxTest’s explicit Inspector edits still apply to its local playback instance. `FxPreview` selects a spell id, with no separate packed-scene/default-duration settings. Adding another implemented spell should extend this catalog/shared playback, not add a second implementation in a preview.

Preview casts use `SpellPresentation.BeginPreviewCast` → `DuelProtocol.PresentationSpell` → `RaceSpriteAnimator.BeginCast`. Live server spell data takes precedence; missing metadata resolves through `SpellPresentationCatalog.Find` (standards delegate to `StandardSpells.Fallback`; Firebolt has a YAML-verified fallback). `fireball` is normalized to `firebolt` for presentation metadata, VFX and preset lookup. Preview code no longer hardcodes 0.75-second casts or a Magic Arrow clip; visual_key/preset resolution remains inside RaceSpriteAnimator. Live cast deadlines still come from the server.

ArenaMapsDev: **Cast gracza** and **Cast przeciwnika** both cast Magic Arrow, and **Rzuć zapisany czar** uses the entered id (initially `magic_arrow`). These play casting, projectile and impact. The separate **Animacja + VFX** controls are deliberately gesture-profile editors, not full spell casts.

## Interfaces and previews

`ISpellEffect` exposes `IsPlaying`, `Stop`, `GetPosition`. `IStaticEffect` adds `Play(position, duration, parameters)`, `OnStarted`, `OnFinished`. `IProjectileEffect` exposes `Play(from, to, duration, parameters)` and `OnFinished`, implemented by Magic Arrow. Area/beam interfaces remain removed.

`Game/ScenesV3/VfxTest/VfxTestScreen.tscn` lists all fourteen catalog effects; projectile previews support position swapping and clearing. `Game/FX/_Previews/MirrorWardPreview.tscn` previews the same shell. `Game/ScenesV3/Dev/ArenaMaps/VerifyMirrorWard.tscn` verifies race attachment, lifetime, directional shatter and live snapshot/event integration.


## Shared projectile lifecycle

`SpellProjectile` owns flight, resolve/dodge, reflection, target tracking, cancellation and completion for both projectile classes. `MagicArrow` and `Firebolt` provide their shader, release decoration and impact/reflection durations. The manager uses the shared catalog `IsProjectile` flag; adding a projectile no longer requires a separate per-spell event/snapshot handler. The factory still dispatches through `IProjectileEffect`. `Resolve` copies `IsProjectile` alongside the other settings.

## Firebolt (Fireball alias)

- Canonical gameplay id `firebolt`; `fireball` remains accepted as a presentation/preset alias. One preset, `Resources/SpellVisuals/firebolt.tres`, replaces the old alias file. ArenaMapsDev save/load normalizes the id before file access.
- Nature Ember (`Tal Rath`), `#FF713D` / `#A52E25`: incandescent sphere, turbulent flame sheets and trailing embers, compact expanding flame impact. Premultiplied coverage masks the blue arena under dense flame while retaining emission in thin wisps. No new audio.
- Server `data/spells/firebolt.yaml`: cast 1 s, recovery 0.4 s, travel 0, mana 9, direct base damage 16. Shared fallback metadata keeps offline preview timing faithful; live server metadata always wins. The 0.18-second cosmetic flight and 0.12-second reflected return do not delay combat. Impact tail is 0.62 s; dodge/cancellation produces no successful-hit burst.
- Cast preset retains `attack_2h_02`, uses the Embers profile, the nature tint, intensity 1.05, scale 0.9, trails and no ground ring.
- In `ArenaMapsDev.tscn`, enter `fireball` or `firebolt`, load the preset, and use **Rzuć zapisany czar** for either side. Ordinary Cast buttons remain Magic Arrow. VfxTest lists Firebolt and uses the same factory for playback, swapping and clearing.
- `Game/FX/_Previews/FireboltPreview.tscn` uses the same `ProjectilePreview` driver as MagicArrowPreview; keys 1/2/3 select hit/reflection/dodge, Tab reverses direction, Space replays. `-- --capture` writes GPU images under `verification/firebolt/` with an explicitly slower 0.30-second inspection flight.
- `VerifyFirebolt.tscn` checks the shared event/snapshot lifecycle, fireball alias and offline timing/preset, both saved-spell controls on the real ArenaMapsDev, catalog overrides and VfxTest tile/clear. Validation 2026-09-08: Firebolt 25/25 checks, Magic Arrow 25/25 and MirrorWard 60/60; build 0 errors / 9 existing warnings. GPU flight/impact/arena captures inspected. Live Nakama and physical Android performance remain untested.

## Magic Arrow presentation

- Nature `arcana`, incantation `Tal Ael`: blue `#64B5FF` and pearl `#EAF4FF`. A faceted lance with three flowing filaments and detached crystalline flecks, a palm-anchored release ring, and a compact broken ring/splinter impact. No fire/smoke layers or new audio.
- The local cast preset keeps `attack_1h_01`, selects Impulse, explicitly sets the Arcana tint, intensity 0.85, scale 0.8 and hides the ground ring. Existing `visual_key` overrides retain precedence.
- Server `data/spells/magic_arrow.yaml` currently has **travel_time 0**. The 0.14-second lance traversal is cosmetic; events, HP, damage logs and combat deadlines are never delayed. A reflected lance returns in 0.10 seconds with a thin afterimage; it keeps Arcana colours. The mirror shatters on its existing authoritative event.
- The manager tracks one arrow per action id. `cast_released` creates it, `spell_impact` resolves its visual ending, and `reason=dodged` fades it without a hit burst. Impact-only delivery can create the missing presentation. `spell_reflected` reuses the arrow with swapped endpoints.
- If a future/pending snapshot contains an arrow, its remaining deadline retimes the flight; repeated snapshots reuse the same node. Missing unresolved pending arrows fade without fabricating an impact. A missing impact event also times out to a fade. Match end/disposal clears tracking, detaches target callbacks and dissolves nodes without starting a new hit burst.
- Source position uses the current casting palm (body centre fallback when idle), target uses the visible actor bounds centre. The effect scales with the actor and tracks its target during flight. The additive analytic shader requires no generated textures or bloom postprocessing.

Preview: open `Game/FX/_Previews/MagicArrowPreview.tscn` and press F6. It loops with cast animation on the real arena; `1` hit, `2` reflection, `3` dodge, `Tab` reverse, `Space` replay. `-- --capture` writes five PNGs to `verification/magic-arrow/`; the flight capture uses 0.30 seconds to make the frame easier to inspect (normal playback remains 0.14 seconds).

Validation: `dotnet build hexbane.csproj --no-restore`; `Godot --headless --path . res://Game/ScenesV3/Dev/ArenaMaps/VerifyMagicArrow.tscn`. GPU preview captures verify shader compilation and visual alignment. Recorded 2026-09-08: build 0 errors / 9 existing warnings; Magic Arrow / shared arena playback 25/25 checks, MirrorWard 60/60 and WardAudio 10/10. Local event/snapshot fixtures cover lifecycle; a live Nakama duel and physical Android performance have not been tested for this effect.

## Source of truth in code
- client:Core/Spells/ISpellEffect.cs
- client:Core/Spells/SpellEffectRegistry.cs
- client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs
- client:Application/Modules/Spell/Effects/SpellEffectFactory.cs
- client:Application/Modules/Spell/Effects/SpellEffectManager.cs
- client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs
- client:Game/ScenesV3/GameHud/ArcaneDuel/Components/SpellAnimationController.cs
- client:Game/FX/MirrorWard.cs
- client:Game/FX/MagicArrow.cs
- client:Game/FX/MagicArrow.gdshader
- client:Application/Modules/Spell/Effects/SpellEffectManager.Projectiles.cs
- client:Game/FX/_Previews/ProjectilePreview.cs
- client:Game/ScenesV3/Dev/ArenaMaps/VerifyMagicArrow.cs
- client:Game/FX/SpellPresentation.cs
- client:Game/FX/_Previews/FxPreview.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud.cs
- client:Game/ScenesV3/Dev/ArenaMaps/ArenaMapsDev.cs
- client:Game/FX/SpellProjectile.cs
- client:Game/FX/Firebolt.cs
- client:Game/FX/Firebolt.gdshader
- client:Core/Spells/SpellPresentationCatalog.cs
- client:Resources/SpellVisuals/firebolt.tres
- client:Game/ScenesV3/Dev/ArenaMaps/VerifyFirebolt.cs

## Actor-bound fields and utility effects — 2026-09-08

`SpellField.cs/.gdshader` implements the eleven newly covered spells from [[spell-vfx-configuration]]. `SpellPresentation` parents non-projectile fields to the selected actor before playback, follows protection bounds and scales with the actor. The canonical resolver/factory serves live duel, ArenaMapsDev saved-spell controls and VfxTest; no view-local spell implementation exists.

Rendering uses two meshes: rear at actor-relative z=0, animated sprite z=1, front z=2. Orbital sections choose their pass from virtual depth; atmospheric body volumes are behind the silhouette, surface glints are in front. The current sprite's alpha naturally masks the rear pass, including idle/cast frames, race changes and mirroring. QuadMesh UV y is inverted explicitly so flames and motes rise and the healing crown appears above the character.

`SpellEffectManager.Fields.cs` maps status kinds `shield → barrier`, `paralyze → paralysis`, and poison/regeneration/delayed_hex directly; reflection retains MirrorWard. `effect_applied` and reconnect snapshots reuse the same instance. Delayed hex uses `due_tick` when provided, otherwise `end_tick`. Snapshot deadlines update the countdown. **Authoritative persistent nodes wait for server removal/snapshot absence**; a local deadline alone never deletes the node before a late damage event. An expired/removed snapshot without damage simply fades and never detonates.

Damage events trigger Heavy Bolt/Consume Venom hits and delayed-hex detonation; heal events trigger Mend/Greater Heal and Regeneration pulses. Poison pulses follow actual damage. Absorbed damage separately finds the target's Barrier (the attacking effect id is not the shield id) and pulses it. Cleanse/Dispel show an execution sweep on non-dodged spell impact; actual status removal remains event/snapshot-owned. Consume Venom without eligible owned poison produces no implosion. Duplicate transient events use a bounded 512-entry identity cache. Completion removes active tracking; cancellation/disposal/match end clears effects without inventing hits. Offline previews simulate periodic pulses and hex detonation for inspection.

`VerifySpellFields.tscn` exercises both real saved-spell buttons for every new spell, target anchors, offline metadata/presets, VfxTest tiles/swap/clear, status/event deduplication, removals, snapshot-only reconnect, delayed authoritative damage, shield absorption and no-op/dodge semantics. GPU-only pixel checks compare rear/front passes over opaque sprite pixels for all six races in both orientations. Use `-- --occlusion-only` for just the pixel suite. Captures live under `verification/spell-fields/` (ignored artifacts).

Source of truth: `client:Game/FX/SpellField.cs`, `SpellField.gdshader`, individual field scenes; `client:Application/Modules/Spell/Effects/SpellEffectManager.Fields.cs`; `client:Game/ScenesV3/Dev/ArenaMaps/VerifySpellFields.cs`; `client:Game/ScenesV3/VfxTest/VfxTestScreen.cs`.

Validation 2026-09-08: build **0 errors / 9 existing warnings**; full GPU `VerifySpellFields` **146/146**, including 24 front/rear pixel assertions for six races × both sides. Adjacent regressions: Magic Arrow **25/25**, Firebolt **25/25**, GPU MirrorWard **60/60**. Two review findings fixed and rechecked (late hex damage, barrier absorption). Rear pass contributes zero measured RGB difference on fully opaque sampled body pixels; front pass visibly changes those pixels. All eleven ArenaMapsDev captures inspected. Live Nakama and physical Android performance remain untested.

Tutorial integration repair (2026-09-08): local `TrainingBattle` now emits canonical `cast_released`, stable per-cast action ids, resolved mentor impacts and status lifecycle ids. It is a producer of the shared manager's events rather than a separate rendering path. The tutorial keeps the arena unobscured for one second after exercise impact before displaying the next explanation. See [[client-tutorial]].
