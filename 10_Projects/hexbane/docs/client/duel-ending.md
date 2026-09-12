---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, duel, animation, results, music]
---

# Duel ending, results and music

## Arena ending

The authoritative result still updates match status and rewards immediately. On the active `MainReference` arena, `SceneManager` leaves the scene in place while its `DuelConclusion` displays the winner and Continue. Other routes without a conclusion still open results directly. There is no protocol change and no delay to damage, XP or persistence.

`Player` starts a terminal death pose on authoritative HP <= 0, including standalone updates. The personalized `defeated` result supplies a fallback when the final snapshot is unavailable: enemy dies on victory, local player on defeat. Timeout, disconnection and surrender do not fabricate a death; a draw only produces corpses when HP says both died. Further cast/idle/meditation messages cannot revive an actor; loading a new actor resets death.

`RaceSpriteAnimator.Die` stops charge, gesture, meditation and casting audio, plays `death` once (25 frames, 20 fps, 1.25 seconds), and holds frame 24. The arena winner panel fades in after 350 ms; Continue is enabled after 1.25 seconds. It consumes arena gestures and combat shortcuts. Any open combat-controls dialog is dismissed without applying its draft, and cannot reopen after the conclusion; this keeps Continue accessible to pointer/touch input. Continue opens the full result screen; the result screen's Continue retains the post-match account refresh/tutorial routing.

## Authored assets

All six races (`human`, `elf`, `dark_elf`, `shadow`, `gnome`, `orc`) have actual skeletal fall actions: recoil, buckling knees, body impact and settling limbs. They are rendered through the existing camera and lights. HD and SD atlases are appended to each race's existing `frames.tres` / `frames_sd.tres`; a wider, symmetrically padded crop keeps the original actor scale and support plane while allowing the horizontal pose. Atlases are 5 by 5 and below 4096 pixels on either axis in HD.

`Resources/Races/_tools/blender/create_death.py` takes the assembled `<race>_mixamo.blend`, authors an editable `death` action, saves a packed source under `Resources/Races/3d/<race>/death/` and renders frames under `animation/death/render/`. `pack_death.py <race>` appends/rebuilds the atlases with atomic PNG replacement (avoids the editor importing a partial file). Source blends and raw renders are excluded from Godot via `.gdignore`; runtime only uses the packed atlases.

## Results hierarchy

`GameOverScreen` has a shared victory/defeat/draw layout in the existing warm brown, cream and gold theme and uses the existing menu backdrop. A small vector seal is fractured on defeat. The primary card emphasizes the outcome, earned XP, level and level-up rewards. The second column retains all three current skill values and gains, record, duel duration, rank-change placeholder, opponent and existing Add friend / Report actions. First-win bonus, XP total / remaining / progress, stat points, MP, new slots, primary tiers and ranked unlock messages remain. Report is still the existing logging-only action; this task does not implement a reporting backend.

Below 1000 logical pixels the cards stack in a scroll view; Continue stays outside that scroll. Large viewports scale the content up to 1.55 times. Data uses native controls, not text baked into a mockup.

## ElevenLabs music

`Resources/Music/results/victory.mp3` and `defeat.mp3` are original instrumental cues generated through ElevenLabs Music v1 using `force_instrumental=true` and 12000 ms prompts. Victory uses a rising orchestral/bright magical palette; defeat a descending cello/piano chamber palette. Both MP3 files are 12.069 seconds including codec padding, 44.1 kHz stereo, about 193 KB each. Prompts, provider, model, song id (when returned), hashes and request options live in `manifest.json`.

Regeneration: `python3 Scripts/Audio/generate_result_music.py`, token only from `ELEVENLABS_API_TOKEN`. Existing files are skipped; no automatic paid retries. Official endpoint reference: https://elevenlabs.io/docs/api-reference/music/compose

At MatchEnded, battle music fades out over 600 ms while the arena remains. Opening the result screen plays the selected cue once via `MenuPlayer`, with no menu/battle overlap, 350 ms fade-in and gain -12 dB victory / -6 dB defeat (draw uses the reflective defeat cue). Music pause/volume settings apply through the existing music path. Leaving results stops the cue and restores menu music.

## Verification

- `DeathVerification.tscn`: six races × HD/SD × both facings, terminal frame, duplicate death, stale cast/idle/meditation, reload. Regression failed before the terminal state existed and passed after implementation.
- `DuelEndingVerification.tscn`: real arena and GameoverHandler, lethal snapshot, winner text, no automatic navigation, delayed Continue, actual pointer navigation, result XP/levelup, 1360×612 / 844×390 / 2400×1080 layouts, victory/defeat/draw and nonlethal timeout, both-dead draw, and an open-controls-modal regression (failed before dismissal fix, passed afterward). Captures under `verification/duel-ending/`.
- `ResultMusicVerification.tscn`: selection, one-shot duration, music mute, no overlapping menu/battle and restoration on exit. GPU/CoreAudio execution records both complete cues to `result-music-reel.wav`; recorded output was 25.228 seconds, non-silent for both, peak -13.48 dBFS. This verifies actual playback, not subjective musical approval.
- Existing meditation transitions/player-state and combat-controls (30 layout/scale/viewport combinations) regressions pass; final C# build has zero errors and nine existing warnings. Godot shutdown retains pre-existing ObjectDB/resource messages.

These are offline fixtures with actual Godot rendering/audio; a live Nakama match and physical mobile device were not tested.

## Source of truth in code

- `client:Game/ScenesV3/Components/RaceSpriteAnimator.cs`
- `client:Game/ScenesV3/GameHud/ArcaneDuel/Components/Player.cs`
- `client:Game/ScenesV3/ReferenceDuel/{ArenaMatch,DuelConclusion}.cs`
- `client:Game/Autoloads/{SceneManager,MenuPlayer}.cs`
- `client:Game/ScenesV3/GameOver/GameOverScreen.cs`, `GameOverScreen.Layout.cs`
- `client:Resources/Races/_tools/blender/create_death.py`, `Resources/Races/_tools/pack_death.py`
- `client:Scripts/Audio/generate_result_music.py`, `Resources/Music/results/manifest.json`
- `client:Game/ScenesV3/Dev/{DeathVerification,DuelEndingVerification,ResultMusicVerification}.cs`
