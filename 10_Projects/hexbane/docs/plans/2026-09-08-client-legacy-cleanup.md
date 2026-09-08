---
type: project
project: Hexbane
area: plans
status: active
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, cleanup]
---
# Client legacy cleanup

User-approved scope: remove old spell VFX, sound effects, storytelling and verified unused dependencies. Preserve current cast animations, Mirror Reflection visuals, current duel gameplay and the icons still used by it. Music is not a sound effect.

1. Remove legacy FX scenes/scripts/assets and audio, reduce the registry and duel presentation to MirrorWard. Keep icon images used by the current catalog.
2. Remove client storytelling RPCs, match manager, DTOs, command/handler, DI and event wiring. Remove retired combat handlers and genuinely unused payloads; retain compatibility DTOs still consumed by development previews.
3. Remove unused tween plugin and stale routes only after checking references.
4. Build, scan all resource references for newly missing assets, run Godot preview/verifier scenes for casting, reflection and current duel icons.
5. Update affected vault notes, decision log and session log with results and remaining limitations.

## Source of truth in code
- client:Game/FX/
- client:Application/Modules/Spell/Effects/
- client:Game/ScenesV3/Components/
- client:Application/EndlessStory/ (removed)
