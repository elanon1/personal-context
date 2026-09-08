---
type: project
project: Hexbane
area: audits
status: implemented
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, client, ui, mobile, verification]
---

# Mobile layout review after the progression redesign

## Findings and changes

Race selection re-created its StyleBox with desktop padding after a compact pass. Its obsolete per-race minimum labels and stat-limit bars also consumed width and height. Selection now preserves fitted padding, shows active traits, and ignores drags; the redundant side summary is hidden in compact mode.

Shared compaction now considers both width and height and scales RichText-specific font overrides as well as Labels. Safe-area margins are applied independently of whether compaction is necessary. Touch targets include clickable primary path cards.

Character detail has a separate tab row on smaller widths and scrollable content for every compact tab. Summary panels and primary cards stack; narrower Stats/Spellbook layouts use one column. Generic BoxContainers allow reflow after every resize, including widening. Summary also tests its actual column minimum widths. Tab and spell selection retain mobile padding.

Dashboard card widths are flexible; content scrolls with the bottom navigation outside it. Cached references preserve asynchronous refresh. Mode selection is compacted. Settings stack their columns on narrower screens. Short lobby screens scroll content while keeping header/footer outside. Short result screens scroll; captions/action targets are larger and long level-up text wraps.

## Verification

`MobileLayoutVerification.tscn` uses offline fixtures without account mutation. It exercises all six races repeatedly, every creation step, all four character tabs, twelve spell cards, two six-tier primary graphs, respec, auth/login/register, dashboard/mode selection, settings, social, news, lobby, populated level-30 game results (including scrolling to the reachable Continue button), and a tutorial instruction card. Checks cover visible outer controls, horizontal clipping inside disabled-horizontal ScrollContainers, and stable race-list width after selection.

- Initial failures reproduced: dashboard overflow at1088×612; dashboard/settings/lobby/results at960×432; creation/dashboard/lobby at1920×1080.
- Final build: `dotnet build hexbane.csproj --no-restore`, zero errors; nine pre-existing warnings.
- Mobile/layout coverage:960×432,1024×768,1088×612,1360×612,1560×720,1920×1080,2400×1080. Final dedicated runs on960×432,1024×768,1360×612,1920×1080 report zero overflows; smaller changes also received rendered inspection.
- Detail resize sequences:1088×612→1360×612→1920×1080, each tab at each size, including starting at1920×1080: zero overflows.
- `Tests/Progression`: PASS (shared budget, combat preview, progression curve, primary paths).
- Existing `StarterSelectionCheck`: PASS (Human4/other3, caps, navigation, invalid choices, exact RPC spell ids).
- Existing `DuelV2Preview` with `HEXBANE_TEST_REFERENCE=1 HEXBANE_TEST_COMPACT=1`: PASS; nine slots, viewport fit, minimum48-unit targets, no overlapping controls, effects and reconnect state.
- Independent static review identified safe-area gating and latched resize handling; both fixed and reviewed again.

Rendered captures are stored under `client:verification/mobile-layout/`. The harness is run with `MOBILE_SIZE=1360x612`; set `MOBILE_CAPTURE=/absolute/path` for PNGs and `MOBILE_RESIZE=1` for detail resize sequences.

## Limits

This is a Godot desktop-rendered simulation of landscape mobile/tablet logical viewports, not a physical Android playtest. Device DPI, keyboard occlusion, actual notch insets and touch feel still need device validation. Authentication failures and ObjectDB shutdown leak warnings are expected from the existing offline scene fixtures. The ordinary control geometry checks do not detect every possible label overlap; rendered screenshots complement them. No live matchmaking session, deployment or gameplay/protocol change was performed. Compaction is applied on screen construction; detail column arrangements additionally react to runtime resizing.

## Source of truth in code

- `client:Game/ScenesV3/_Common/ResponsiveLayout.cs`
- `client:Game/ScenesV3/CreateCharacter/CreateCharacterScreen.cs`
- `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.cs`, `.Responsive.cs`, `.Primary.cs`
- `client:Game/ScenesV3/Dashboard/DashboardScreen.cs`, `ModeOverlay.cs`
- `client:Game/ScenesV3/Settings/SettingsScreen.cs`, `Lobby/LobbyScreen.cs`, `GameOver/GameOverScreen.cs`
- `client:Game/ScenesV3/Dev/MobileLayoutVerification.cs`, `.tscn`

## Follow-up: visible, touch-friendly arcane scrollbars

Implemented shared amber/gold runic bars with 48-unit input lanes, 24-unit visible thumbs and a subtle animated glow/glint. Restored bars previously hidden by Dashboard/News/Social/Lobby. Adjusted creation summary, Social and Lobby below 1100 wide for the additional lane width.

Validation: build 0 errors / 9 existing warnings; ScrollbarVerification PASS for native mouse drag, synthetic touch drag including the gutter, visible 48-unit target and minimum thumb length for very long content. MobileLayoutVerification reports 0 overflows at 960×432, 1088×612, 1360×612 and 1920×1080. Actual GPU rendering at 1360×612 produced 55 animation frames; scroll value stayed unchanged and frame pixels vary. Screenshots inspected; code review found no definite regressions. Evidence: client `verification/arcane-scrollbar/`. No physical Android validation or deployment was performed.
