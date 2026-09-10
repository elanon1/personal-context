---
type: project
project: Hexbane
area: plans
status: complete
created: 2026-09-09
updated: 2026-09-09
verified: 2026-09-09
tags: [hexbane, ui, keyboard, desktop, mobile]
---

# Desktop combat HUD and keyboard controls

Goal: adapt the active ReferenceHud to PC while retaining a touch layout, with persistent configurable keys and visible bindings.

Architecture: one combat presenter and command path, two layout profiles (Auto/Desktop/Mobile). Auto uses OS platform, never window width. Window size controls fitting, not input mode. A shared controls dialog edits a draft configuration and saves only on Apply; the same dialog is accessible from Settings and the duel. Physical keyboard bindings remain independent of race/loadout: seven draft positions, Magic Arrow, Mirror Reflection, meditate and clear queue. Defaults 1–5/R/F, Q/E, Space/X. Escape opens/closes the modal, cancels capture; Delete/Backspace unbind. Reject conflicts and modifier chords. Existing touch interactions and protocol stay intact.

- [x] Add settings model with validated key persistence, profile and HUD scale; conflict handling and reset.
- [x] Add shared scrollable controls dialog, visible conflict errors, Apply/Cancel, and settings-screen entry.
- [x] Add desktop action bar using existing warm palette, shared fonts and live spell textures; keycaps below slots. Preserve mobile arrangement, safe area and tutorial hooks. Resizing recomputes geometry; desktop scale is clamped to available space.
- [x] Route keyboard input through the same guarded cast/meditate/clear functions as buttons; block gameplay behind dialogs and reject echo/modifiers.
- [x] Build and run Godot verification: saved settings, collisions, cancellation, default reset, stable primary keys, PC/mobile fitting, resizing, screenshots and existing duel smoke.
- [x] Update client contracts, decision log and session journal with evidence and limits.

Validation uses offline Godot scenes and synthetic input (no account writes). Physical device ergonomics and live networking remain separate manual checks. Do not commit unrelated pre-existing workspace changes.

## Source of truth in code
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud*.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceSpellSlot.cs
- client:Game/ScenesV3/Settings/CombatControls.cs
- client:Game/ScenesV3/Settings/CombatControlsDialog.cs
- client:Game/ScenesV3/Settings/SettingsScreen.cs


## Completion evidence (2026-09-09)

Build: 0 errors, 9 pre-existing warnings. Rendered Godot acceptance passes all 30 layout/scale/viewport combinations and input/config/focus/race/tutorial/preset checks. Existing duel snapshot/effect/queue/reconnect smoke passes Human at 2400×1080 and Elf at 1360×612. Core tutorial tests pass. Review findings on focus confinement and desktop tutorial targeting resolved and re-reviewed; real HUD layer ordering and resource-track layering covered by regression checks. Screenshots/logs: client:verification/combat-controls/. Physical device touch/DPI and complete online duel are not verified. Existing DuelV2 smoke retains ObjectDB shutdown leak warnings.
