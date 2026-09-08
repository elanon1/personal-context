---
type: project
project: Hexbane
area: plans
status: complete
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

## Verification results

- `dotnet build hexbane.csproj --no-restore`: 0 errors, 9 warnings (baseline: 37 warnings).
- `VerifyMirrorWard.tscn`: 60 checks, 0 failures, including absence of legacy projectile/impact VFX.
- `VerifyGestureVfx.tscn`: 8899 checks, 0 failures. Corrected an outdated test that expected generic hand charge even with a custom cast preset; runtime casting code was preserved.
- `DuelV2Preview.tscn` with `HEXBANE_TEST_REFERENCE=1`: passed active HUD layout/state/reconnect UI and all 14 icons. Offline startup still logs a NotificationModule null-reference and ObjectDB exit leaks; no live Nakama match or Android run was performed.
- Resource-reference scan: no remaining literal resource references to any of the 863 removed files (287.5 MiB).
- Recovery copies (including untracked files): `/tmp/hexbane-removed-legacy.tar.gz`, `/tmp/hexbane-removed-code.tar.gz`, `/tmp/hexbane-removed-pipeline.tar.gz`; pre-task tracked diff: `/tmp/hexbane-before-cleanup.patch`.
- Client repository only: backend storytelling removal is not part of these code changes.
