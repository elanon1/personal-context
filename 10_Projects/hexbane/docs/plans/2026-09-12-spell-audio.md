---
type: plan
project: Hexbane
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
status: implemented
---
# Spell audio implementation

User request: generate natural, pleasant ElevenLabs sounds matched to all spell visuals and integrate the sound system.

Design: extend existing Game/FX lifecycles. Scene-owned one-shots preserve decays after short VFX disappear; actor-owned casting sounds fade on interruption. Keep combat timing and network messages unchanged. Central SpellAudio caches streams, caps simultaneous voices at 16 and identical streams at three. Materials follow YAML nature and the actual shader geometry. Shared SpellFX bus provides short damped reverb, high-frequency filtering and output limiting.

- [x] Inspect all 14 YAML identities and existing projectile, field and ward lifecycle paths.
- [x] Generate 21 event cues and five nature casting textures via ElevenLabs Sound Effects v2. Preserve prompts and hashes in Resources/Audio/Spells/manifest.json. Repeat generation only explicitly for missing files.
- [x] Add a failing shared-factory/lifecycle verifier; baseline: 35 assertions, 31 failed (missing spell audio).
- [x] Integrate SpellAudio into SpellProjectile, SpellField, MirrorWard/WardAudio and RaceSpriteAnimator. Configure bus processing.
- [x] Build, import assets and run audio lifecycle plus adjacent spell verifiers.
- [x] Deliver an audition recording and record subjective listening limitations.
- [x] Update runtime notes, index, decisions and session log.

Validation focuses on cancelled/dodged projectiles producing no hit, once-only hex detonation, removed statuses never pulsing, tails surviving visuals, actor disposal and voice cap. Audio measurements are not a subjective naturalness judgment. Physical mobile and live Nakama listening require separate verification.

Results: build 0 errors / 9 existing warnings; audio 40/40, ward 10/10, fields 122/122; Magic Arrow and Firebolt zero failures. CoreAudio bus recording 57.48 s, all cues non-silent, sample peak -10.81 dBFS. Static review found no important bugs. Casting texture is a ~2 s one-shot; longer modified casts can have a quiet gap. Subjective headphone/mobile/live-duel listening remains unverified. Existing resource-at-exit warnings also occurred in the preimplementation baseline.
