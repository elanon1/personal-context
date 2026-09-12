---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, dev, spells, vfx, audio]
---

# Gameplay presentation sandbox

Open `Game/ScenesV3/Dev/GameplaySandbox.tscn` and run the current scene (F6). This is an offline presentation workbench, explicitly requested for visual/audio inspection and manually controlled scenarios. It does not resolve combat damage, mana, status eligibility, progression or network messages.

The card grid comes from `SpellEffectConfigurations.GetDefaultConfigurations()`, maps the presentation alias `mirror_ward` to `mirror_reflection`, and loads icons through `Spell.GetTexture`. All 14 current spells are selectable. New configurations automatically add cards; their shared presentation metadata and icons must also exist. The library scrolls vertically if it grows.

## Controls

- Select a spell icon, choose the casting side, then **Rzuć / Space**. Shared `SpellPresentation.BeginPreviewCast` and `SpellPresentation.Play` provide race gestures, cast audio, anchoring, VFX and SFX. No scene-local effect implementation or copied effect duration table.
- Choose either side's race independently (six current races), or one of the four shared arena maps. Changing race resets playback first.
- **Trafienie / Odbicie / Unik** select projectile presentation for Magic Arrow and Firebolt. Reflection creates a target mirror and returns the projectile; dodge does not generate a hit burst. The outcome picker is disabled for non-projectile effects.
- **Powtarzaj** repeats the current spell/side after release and recovery. Both sides can cast independently. Active effects are capped at 32 to bound held-status accumulation.
- **Utrzymaj status** applies to subsequent casts: persistent fields remain until cleared/detonated and do not synthesize periodic ticks. Normal playback simulates pulses and expiration as in the existing previews. Mirror duration remains the shared catalog duration.
- **Medytuj / M**, **Stop medytacji**, **Przerwij / X** operate on the selected caster. Meditation uses the actual race animation, effect and shared audio loop. Interruption cancels a pending cast release and stops the automatic repeat.
- **Impuls statusów** pulses fields on both sides; **Detonuj hex** detonates active delayed-hex fields; **Rozbij lustra** shatters active mirrors.
- **Wyczyść wszystko / R** cancels both pending casts and loops, removes all spell effects, stops spell voices and returns both actors to idle. It preserves spell/map/race selection.
- **Panel / Tab** hides the spell dock for unobstructed inspection. **Spokojne tło** reduces arena motion. Mute and volume operate on SpellFX (Master fallback); the previous mix is restored when leaving the scene.

Useful manual cases: hold poison and regeneration on the same actor, pulse then clear; hold a delayed hex then detonate or clear before detonation; reflect a Firebolt in either direction; start Heavy Bolt then interrupt before release; start meditation then cast; change race while a cast is pending.

These are visual fixtures: e.g. manually showing healing on a poisoned actor does not imply the server would permit that heal. Use the live duel/server tests for gameplay semantics. Existing ArenaMapsDev remains the gesture/preset editor; VfxTest remains the effect configuration editor.

## Verification

`GameplaySandboxVerification.tscn` exercises viewport fit, all 14 icons and actual shared casts, pending-cast reset cancellation, reverse-direction reflection/dodge, meditation animation/audio/stop and a held poison beyond its normal lifetime. GPU run on 2026-09-12 passed all checks; the final capture was visually inspected after fixing inherited theme padding that had pushed buttons off screen. Build: 0 errors, 9 existing warnings. With a rendering backend and `-- --capture`, it writes `/tmp/hexbane-gameplay-sandbox.png`. Headless runs cannot establish shader appearance or subjective audio quality. Live Nakama and physical devices are outside this offline scene's verification scope.

## Source of truth in code

- client:Game/ScenesV3/Dev/GameplaySandbox.cs
- client:Game/ScenesV3/Dev/GameplaySandbox.tscn
- client:Game/ScenesV3/Dev/GameplaySandboxVerification.cs
- client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs
- client:Core/Spells/SpellPresentationCatalog.cs
- client:Game/FX/SpellPresentation.cs
- client:Game/FX/SpellAudio.cs
