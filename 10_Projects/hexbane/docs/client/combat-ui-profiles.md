---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-09
updated: 2026-09-09
verified: 2026-09-09
tags: [hexbane, ui, desktop, mobile, keyboard]
sources: ["client:Game/ScenesV3/ReferenceDuel/ReferenceHud.Layout.cs", "client:Game/ScenesV3/Settings/CombatControls.cs"]
---

# Combat UI profiles and keyboard controls

The active duel uses one presenter and two layouts. No platform-specific generated art, duplicate networking or separate game export is needed. Both PvP and AI use `MainReference.tscn`; the obsolete GameHud preview is not the implementation target.

## Player settings

Open **Settings → Interface → Keyboard & combat HUD…**, or **Esc / Controls** during a duel (gear on mobile). The modal edits a copy; **Apply** saves, **Cancel** discards, **Defaults** resets bindings, profile and PC scale in the draft. An online duel keeps running. The existing master-volume adjustment is available here and applied for the session on Apply; persisted audio preferences remain in the general Settings screen.

- **Automatic**: Android/iOS/mobile feature → Mobile; otherwise PC, including macOS/Linux. Detection uses platform, not window width, so a narrow desktop window keeps its keyboard layout.
- **PC / Mobile**: explicit override, usable on either platform for preview/testing.
- **PC HUD size**: 85–115%, in 5% steps, affects the bottom action bar. Fitting retains usable targets and readable keycaps at compact sizes. Mobile ignores this slider.
- Click a binding, press one key. Physical positions are persisted; visible legends reflect the current OS keyboard layout (refreshed on viewport resize/focus and preferences changes). Esc cancels capture, Delete/Backspace unbind, modifier chords are rejected. A conflict identifies the existing action; clear that assignment first. Swapped keys survive reload.
- The modal acquires focus, confines Tab/arrow navigation and restores the prior control on close; the dimmer stops pointer input. Its layer is above the real duel HUD. Gameplay input is gated while it is open.

| Action | Default |
|---|---|
| Draft slots 1–5 | 1–5 |
| Draft slot 6 | R |
| Draft slot 7 (Human) | F |
| Magic Arrow | Q |
| Mirror Reflection | E |
| Meditate | Space |
| Clear queue | X |

These are positional draft bindings, not spell IDs. Lobby slot ordering determines the spell in each position. Primary actions remain Q/E across Human/non-Human capacity changes. Absent slot 7 does nothing for non-Human characters. Old fixed 1–9/Space/M mappings in ReferenceHud are superseded.

`CombatControls` persists to **user://combat-controls.cfg** (separate from general `settings.cfg`). Invalid types, invalid keys or conflicting stored keys restore safe default bindings; profile and scale are bounded. Save errors remain visible and do not change the active configuration. Settings are per local installation, not synchronized to an account. Single keyboard keys only; mouse-button bindings, chords and gamepad remapping are outside this version.

## Layout and rendering

PC: warm dark resource panels, a centered bottom action bar, separate draft and primary groups, native spell icons, mana costs, rectangular targets, cast progress, queue/hover accent and keycaps beneath every slot. Mobile: original touch rails, circular slots and prominent primary spells, no keyboard labels. Both consume the same spell/loadout/snapshot state and guarded cast/meditate/clear-queue paths. Key repeats and modified system shortcuts cannot cast.

`ReferenceHud.FitCanvas` retains the existing safe-area transform and arena layout. `ReferenceHud.Layout.cs` owns desktop chrome, group geometry, controls labels and profile selection; `LayoutSlots` retains mobile geometry. `ReferenceSpellSlot.SetPresentation` changes only rendering/hit geometry. The theme uses existing warm palette, Inter text and Cinzel group caps. Resource tracks render behind the live fills. Resource amounts and combat state remain authoritative.

The local tutorial selects the corresponding input policy. Desktop uses configured shortcuts for the allowed target; paused/non-target slots remain gated. The loadout highlight merges actual desktop draft rectangles. Guided training does not open the combat settings modal; configure bindings from Settings before replay. Touch input and mobile swipe meditation remain available.

## Development process for distinct PC/mobile UI

1. Open `Game/ScenesV3/Dev/DesktopCombatPreview.tscn` or `MobileCombatPreview.tscn` and run with F6. Both inherit the same offline reference HUD and force their profile through exported **LayoutOverride**. They do not log in or send match commands. PreviewCapacity/PreviewUnlocked/PreviewMana/PreviewSafeInsets allow fixture variations.
2. Change shared spell positions in `CombatSlotGeometry.Slots`; PC chrome remains in `ReferenceHud.Layout.cs` → `LayoutDesktopSlots` / `LayoutPresentation`, mobile chrome in `ReferenceHud.cs` → `LayoutSlots`. Shared slot styling belongs in `ReferenceSpellSlot` and shared tokens in ScenesV3 themes. Keep snapshot handling and cast commands out of profile branches.
3. Use **Auto** on the production HUD; platform detection selects the profile at runtime. A player's explicit preference overrides auto, and an exported preview override takes priority for that preview.
4. Resize previews and check both modes after shared-component changes. Do not infer Mobile from a small desktop window. The supported combat composition is landscape; portrait gameplay needs its own design.
5. Build, run the offline acceptance scene, and render screenshots using the commands below. Update this note/design-system contract when behavior changes.

```sh
dotnet build hexbane.csproj --no-restore
HEXBANE_IGNORE_ENV_FILE=1 /Applications/Godot.app/Contents/MacOS/Godot --headless --path . res://Game/ScenesV3/Dev/CombatControlsVerification.tscn
HEXBANE_IGNORE_ENV_FILE=1 COMBAT_CAPTURE="$PWD/verification/combat-controls" /Applications/Godot.app/Contents/MacOS/Godot --path . res://Game/ScenesV3/Dev/CombatControlsVerification.tscn
```

The verifier uses offline fixtures, synthetic key input and config round trips, restores the user's original config in `finally`, and checks modal focus, save/cancel/conflicts/defaults, echo/modifier/paralysis gates, stable primary keys, tutorial targets, preview selection and 30 profile/scale/viewport combinations: 1920×1080, 1360×612, 1280×720, 2560×1080 and 960×432 at 85/100/115%. Do not run two preference-writing verification instances concurrently.

Validation 2026-09-09: build 0 errors / 9 existing warnings; rendered acceptance and Core tutorial tests PASS; Human 2400×1080 and Elf 1360×612 DuelV2 smoke PASS (existing ObjectDB shutdown warnings). Review findings resolved.

Evidence: `verification/combat-controls/` PNGs (PC 1080p, compact PC/mobile, controls dialog) and logs. These verify desktop rendering and synthetic input, not physical Android/iOS touch, actual DPI or a full online duel. Existing DuelV2Preview additionally exercises snapshots, effects, queue and reconnect controls without a server; Core tutorial tests cover training mechanics. No match protocol or balance changes.

## Pre-match spell arrangement (2026-09-09)

After the draft fills, `LobbyScreen.EnterArrangementMode` hides draft chrome and shows `SpellArrangementView` over the existing shared UI background. It uses **ReferenceSpellSlot** and **CombatSlotGeometry.Slots**, the same components/geometry used by the duel. PC shows rectangular slots and current keycaps; mobile shows the same circular left/right arrangement and fixed primary pair. Resource bars, actors, portraits and combat actions are absent.

- Drag a populated draft slot onto another populated draft slot to swap them. Positions retain their key bindings. Empty capacity slots cannot receive spells; primary slots remain fixed and do not accept drops.
- Mouse and native touch are handled directly, without relying on mouse emulation. A 12-unit movement threshold distinguishes tapping from dragging. Release outside, native touch cancellation, Escape, viewport resize and window focus loss discard the pending drag. A translucent icon follows a valid drag, and source/target highlights identify the swap.
- Click/tap any populated slot, including either primary, to inspect its description. A swap selects the moved spell in its destination. The card shows the loaded spell's mana cost and cast time; long text scrolls independently. At small mobile sizes the card occupies the gap between the spell groups to remain readable.
- Primary metadata comes from the pending authoritative match loadout when available. Until then primary slots keep their baseline identity/description, omit numeric mana/timing and explain that character-specific values/effects are pending. Draft metadata missing from the loaded library is treated similarly; do not imply zero cost or zero cast time.
- Auto Sort orders draft spells by mana then name. Ready keeps the existing lobby handler and locks further swaps/sorting while retaining inspection. The existing timer still updates. Swaps/sort immediately save `MatchContext.SpellSlotOrder`; the duel's existing loadout ordering consumes it. No protocol or ready-message changes.

Development: change shared slot geometry in `ReferenceDuel/CombatSlotGeometry.cs`, then verify both combat and arrangement. Arrangement-specific layout/inspection/drag logic lives in `Lobby/SpellArrangementView.cs`; `LobbyScreen` only supplies data, order persistence, timer and Ready wiring. The old alternating LEFT HAND/RIGHT HAND card implementation and its mouse-only drag handler were removed.

```sh
dotnet build hexbane.csproj --no-restore
HEXBANE_IGNORE_ENV_FILE=1 ARRANGE_CAPTURE="$PWD/verification/spell-arrangement" /Applications/Godot.app/Contents/MacOS/Godot --path . res://Game/ScenesV3/Dev/SpellArrangementVerification.tscn
```

Offline acceptance uses real synthetic mouse/native-touch events, validates swaps, inspection, canceled/outside/fixed-primary drops, actual HUD order handoff and Ready lock. It renders both profiles at 1920×1080, 1360×612 and 960×432, asserting no clipped/overlapping slot/detail rectangles and readable description height. It also enters the view via actual LobbyScreen selection events, checks saved order, updates the countdown and invokes the existing Ready handler. Screenshots/logs: `verification/spell-arrangement/`. Physical-device ergonomics and a full online draft→duel remain manual validation.

## Source of truth in code
- client:Game/ScenesV3/Lobby/SpellArrangementView.cs
- client:Game/ScenesV3/Lobby/LobbyScreen.cs
- client:Game/ScenesV3/ReferenceDuel/CombatSlotGeometry.cs
- client:Game/ScenesV3/Dev/SpellArrangementVerification.cs
- client:Game/ScenesV3/Settings/CombatControls.cs
- client:Game/ScenesV3/Settings/CombatControlsDialog.cs
- client:Game/ScenesV3/Settings/SettingsScreen.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud.Layout.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud.DuelV2.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceSpellSlot.cs
- client:Game/ScenesV3/Tutorial/TutorialControls.cs
- client:Game/ScenesV3/Tutorial/TutorialScreen.cs
- client:Game/ScenesV3/Tutorial/TutorialOverlay.cs
- client:Game/ScenesV3/Dev/CombatControlsVerification.cs
- client:Game/ScenesV3/Dev/DesktopCombatPreview.tscn
- client:Game/ScenesV3/Dev/MobileCombatPreview.tscn
