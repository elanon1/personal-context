---
type: project
project: Hexbane
area: client
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
status: active
---
# Spell audio

26 ElevenLabs Sound Effects v2 recordings in `Resources/Audio/Spells/`: 21 spell events and five quiet casting textures. The manifest preserves prompts, duration, provider, request identifiers where returned, sample peak/RMS and SHA-256. Actual materials follow [[spell-lore]] and the existing shaders: small glass lance for Magic Arrow, flame for Ember, warm silk/wood/air for Vitality, viscous liquid for Venom, tensile wood/leather/paper for Hex. No spoken incantations are generated.

## Runtime

`Game/FX/SpellAudio.cs` caches streams and owns one-shot playback. All sounds route through `SpellFX` with Master fallback. One-shots belong to the current scene, so material tails survive the short visual. Casting belongs to its actor and fades on FinishCast/rest or disappears with actor disposal. Casting textures are short one-shots (about 2 seconds), not continuous loops; a longer modified cast can have a quiet gap before release. At most 16 voices and three of one stream play concurrently; further requests are dropped. Small ±2% pitch variation reduces identical repetition without changing spell identity.

`SpellProjectile` plays release at launch and impact only when its visual enters a resolved non-dodged burst. Stop/dodge never emits impact, and fades the launch sound. Reflection retains the original sound material. These are cosmetic deadlines; no gameplay or network timing changed.

`SpellField` starts a per-style cue, pulses only when its existing pulse callback fires (actual damage/heal/absorption in authoritative play), and plays delayed-hex rupture in Detonate only once. Removed persistent fields fade their starting voice, and never emit an expiration explosion. Transient heal/cleanse/strike tails are allowed to finish after the visual. Existing event/snapshot deduplication is reused.

`MirrorWard` loads new ElevenLabs formation/rupture assets through SpellAudio, preserving exported AudioStream overrides. WardAudio retains its compatibility group and routes through the shared voice budget. Old `Resources/Spells/mirror_reflection/sfx` files remain available but are no longer default runtime streams.

## Mix and asset preparation

Generator: `python3 Scripts/Audio/generate_spell_audio.py --generate` (macOS afconvert, Python numpy; API token only from `ELEVENLABS_API_TOKEN`). Without `--generate`, only reports missing cues. Existing generated cues are skipped; `--only spell/stage` selects one cue. No automatic paid retries. Generation uses the official `POST /v1/sound-generation` API and model `eleven_text_to_sound_v2`.

Downloaded audio is decoded into stereo 44.1 kHz 16-bit WAV, DC corrected, leading silence trimmed, and given 3 ms attack / 50 ms tail fades. Gain aims at -21 dBFS full-file RMS subject to a -3 dBFS sample-peak ceiling, preserving strong transients. This is not LUFS or true-peak normalization. Current total: 37.242 seconds, 6.57 MB PCM.

`default_bus_layout.tres`: short damped room (size .32, damping .72, wet .12, dry 1, predelay 18 ms), low-pass 12.5 kHz, hard limiter ceiling -1 dB. Casts sit at -13 dB, pulses -11 dB, light projectile below heavy strike. The limiter bounds the spell bus; it does not limit the summed music + SFX Master output.

## Try and verify

Rebuild and restart Godot scenes after C# changes. In `Game/ScenesV3/Dev/ArenaMaps/ArenaMapsDev.tscn`, enter a spell id and use **Rzuć zapisany czar** for either actor. `Game/ScenesV3/VfxTest/VfxTestScreen.tscn` uses the same factory. Gesture-only editor controls retain their meaning.

`SpellAudioAudition.tscn` records the actual bus and saves `verification/spell-audio/spell-audio-reel.wav` plus `timeline.json`. Run with CoreAudio (not the headless dummy audio driver) for a real mixed recording. Order follows SpellEffectConfigurations: Magic Arrow, Firebolt, Mirror Reflection, Heavy Bolt, Delayed Hex, Poison, Paralysis, Cleanse, Mend, Greater Heal, Regeneration, Barrier, Dispel, Consume Venom.

Verification: `VerifySpellAudio.tscn` checks factory playback for all 14 spells, routing, cancellation/dodge, once-only hex, removed status pulses, tail completion, actor disposal and voice cap. The baseline before implementation failed 31/35; implementation passed 40/40. Ward audio regression passed 10/10. Magic Arrow and Firebolt regressions also passed (zero failures). The actual CoreAudio recording is 57.48 seconds, stereo, non-silent for each spell, with maximum sample peak -10.81 dBFS. Further regression results are recorded in the session log.

These checks establish playback and signal properties, not subjective naturalness. Human headphone/speaker listening, live Nakama mix and physical mobile playback remain unverified.

Source: https://elevenlabs.io/docs/api-reference/text-to-sound-effects/convert


## Meditation audio — 2026-09-12

Additional ElevenLabs `meditation/loop.wav` (7.75 seconds after 250 ms overlap preparation; manifest includes `loop: true`) brings the library to 27 recordings. Natural flowing air, rubbed glass and silk express the blue inward absorption trails without a melody or pulse announcing mana ticks. The existing spell bus provides the room response. The raw loop has no attack/tail fades; crossfading its ends preserves a continuous wrap, and runtime volume handles entry/exit.

`MeditationVfx.SetPose` starts an owned voice only on inactive→active, with 350 ms fade to -16 dB and fixed pitch. `Clear` kills the entrance tween and fades the voice over 180 ms. Repeated pose/snapshot updates reuse the voice. Casts, rest, race reload and state interruption already call Clear, and deleting the actor deletes the looping player. `SpellAudio.Load` configures WAV cues with stage `loop` for forward looping over the whole sample. Existing one-shot behaviour is preserved.

Verification: `MeditationAudioTest.tscn` failed before integration (no meditation voice), then passed 8/8 using actual CoreAudio: start, deduplication, continued playback past sample duration, cast interruption, resumption, rapid restart, stop and actor disposal. Spell audio regression passed 40/40; build 0 errors / 9 existing warnings. `-- --record` saves `verification/spell-audio/meditation-demo.wav` with actual bus processing. Boundary discontinuity .000916 full scale, below the largest interior sample step .00851; source peak -11.7 dBFS. Human listening and physical mobile remain unverified.

Try `Game/ScenesV3/Dev/MeditationVfxPreview.tscn` (M meditate, X stop), or meditation in the duel. It uses the same MeditationVfx lifecycle.
