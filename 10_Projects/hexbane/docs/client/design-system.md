---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, client, ui, design-system, themes]
sources: ["client:docs/DESIGN_SYSTEM.md"]
---

# Design system (ScenesV3)

Reference for building ScenesV3 screens in the existing style. Theme files live in `Game/ScenesV3/_Themes/` (the old doc said `_Themes/` at the root; that path does not exist):

| Theme | Applied to | Type variations |
|---|---|---|
| `m_auth_theme.tres` | base theme (Inter defaults: Label Medium, Button SemiBold, LineEdit/RichTextLabel Regular); no variations | — |
| `m_auth_card_theme.tres` | the floating `Card` on `AuthScreen.tscn` | `TitleLabel`, `SubtitleLabel`, `CapsLabel`, `DividerLabel`, `FooterLabel`, `FooterLink`, `PrimaryButton` |
| `m_charcreate_theme.tres` | every other ScenesV3 screen: Auth, CreateCharacter, CharacterDetail, Dashboard (+ `ModeOverlay`), Lobby, News, Settings, Social, GameOver | `CapsLabel, ChipButton, CircleButton, CircleButtonAccent, DetailTitle, GhostButton, NavButton, NavButtonDanger, OrnateButton, PlaqueButton, PlayButton, PrimaryButton, RaceCardName, SearchInput, SideTitle, SideValue, SpellName, StatRowPanel, StatSlider, StepCaption, StepName, SubheadLabel, SubtitleLabel, TabButton, TitleLabel` |

Reuse these variations instead of per-scene styleboxes. `TabButton`'s active state is a `StyleBoxFlat` built in code. Shared helpers: `Game/ScenesV3/_Common/` (`ResponsiveLayout` compact mode for the ~1360×612 phone viewport, `UiKit`).

## Fonts (`Resources/Fonts/`, static TTFs only, wired in the themes, never per node)

| Role | Family | Wiring (verified in the `.tres` files) |
|---|---|---|
| Caps headers, caps buttons, plaques, PLAY | **Cinzel** SemiBold (PLAY: Bold), `spacing_glyph 3` (PLAY 6) | `font_caps`/`font_play` sub-resources → `CapsLabel, OrnateButton, NavButton, NavButtonDanger, PlaqueButton, PlayButton` (`m_charcreate_theme.tres:382-521`); auth card `CapsLabel`, `DividerLabel`, `PrimaryButton`; GameOver VICTORY/DEFEAT load `Cinzel-Bold`/`Cinzel-Black` in code (`GameOverScreen.cs:40-41`) |
| Character names, section names, spell names | **Cormorant Garamond** SemiBold, `spacing_glyph 1` | `font_title` → `TitleLabel`, `DetailTitle`, `RaceCardName`, `SpellName`; auth tagline |
| Sub-headers in the auth card | **Marcellus** Regular | only `SubtitleLabel` in `m_auth_card_theme.tres:201` |
| Everything else | **Inter** Regular / Medium / SemiBold (`Inter_24pt-*`) | theme defaults; `SubheadLabel`, `SideTitle`, `StepName`, `TabButton`, `ChipButton` use Inter **Medium** (`m_charcreate_theme.tres:559-604`), `SubtitleLabel` Inter Regular, `PrimaryButton` SemiBold |

The old doc claimed `SubheadLabel`, `SideTitle`, `StepName` and `TabButton` were Marcellus; the theme wires Inter Medium. Cormorant has a small x-height: size it ~15 % larger than adjacent Inter text (spell rows 24–26). Keep Cinzel caps at 18–26.

## Background

Every ScenesV3 screen and GameOver use `Resources/Images/Backgrounds/ui_background.png` (9 scene/script references) under a dark `BgOverlay` panel. The duel scene has its own arena stage ([[vfx-and-race-animation]]). Do not generate per-screen backgrounds. Auth-only assets: `Resources/Images/Auth/` (ornament, sigil, icons, `button_bg.png` 9-patch).

## Palette (warm-toned: R > G > B in every dark, never grey or blue-black)

| Token | Color | Use |
|---|---|---|
| Accent | `Color(0.85,0.35,0.12)` `#D9591F` | buttons, focus borders, active step, selected cards, links |
| Accent hover / pressed / disabled | `(0.95,0.42,0.18)` / `(0.72,0.28,0.08)` / `(0.3,0.15,0.1,0.6)` | button states |
| Text primary / label / input / placeholder | `(1,1,1)` / `(0.92,0.87,0.8)` / `(0.95,0.92,0.88)` / `(0.65,0.58,0.5,0.8)` | |
| Muted / field label / description / disabled | `(0.72,0.66,0.6,0.85)` / `(0.75,0.7,0.65,0.9)` / `(0.6,0.55,0.5,0.6)` / `(0.5,0.5,0.5)` | |
| Error / warning / success / negative / neutral | `(0.95,0.3,0.25)` / `(0.9,0.8,0.3)` / `(0.4,0.85,0.4)` / `(0.85,0.4,0.4)` / `(0.6,0.6,0.6,0.8)` | validation, points counters, modifiers |
| STR / INT / DEX | `(0.85,0.45,0.35)` / `(0.35,0.55,0.85)` / `(0.35,0.8,0.45)` | stat labels and values |
| Spell meta: school / mana / cast time | accent at 90 % / `(0.4,0.6,0.9,0.9)` / `(0.7,0.65,0.6,0.8)` | |
| Step indicator active / completed / inactive | accent / `(0.7,0.65,0.6,0.9)` / `(0.4,0.35,0.3,0.6)` | |
| Screen overlay | `(0.02,0.01,0.008, 0.45–0.55)` | tint over the background image |
| Panels: form / content / detail / glass / card | `(0.04,0.02,0.015,1)` / `(0.06,0.03,0.02,0.7)` / `(0.04,0.02,0.015,0.8)` / `(0.08,0.04,0.03,0.82)` / `(0.08,0.04,0.03,0.85)` | |
| Inputs normal / focus | `(0.06,0.04,0.03,0.9)` / `(0.08,0.05,0.04,0.95)` | |
| Slots empty / filled | `(0.04,0.02,0.015,0.6)` / `(0.1,0.06,0.04,0.9)` | |
| Borders subtle / very subtle / input / input focus / slot empty / selected / chosen | `(1,1,1,0.08)` / `(1,1,1,0.06)` / `(1,1,1,0.15)` / accent 90 % / `(1,1,1,0.1)` / accent / `(0.4,0.85,0.4,0.8)` | |

Accent literal appears in 5 scene/theme/script files; the remaining values were carried over from the 2026-09-02 rework and were spot-checked, not re-measured (unverified individually).

## Geometry

| Item | Values |
|---|---|
| Corner radii | 8 inputs and spell slots; 10 buttons, detail panels, cards; 12 content/glass panels |
| Border widths | 1 panels; 2 inputs, filled slots, chosen cards; 3 selected/focused cards |
| Container separation | 2 tight stacks · 4 card internals · 6 form groups · 8 header groups · 10 lists · 12 slot bars · 16 primary layout · 20 step content · 40 summary columns |
| Theme padding | Button 32/16/32/16, LineEdit 20/14/20/14, glass PanelContainer 36/32/36/32 |
| Scene padding | content panel 32/24, detail panel 16/12, compact auth panel 24/16; cards 12, slots 8 all sides |
| Screen outer margins | 60 / 30 / 60 / 20 (L/T/R/B) |
| Font sizes | 14 captions · 15 spell meta · 16 modifier rows · 18 step/field caps/errors · 20 section sub-headers · 22 default/values/inputs · 24 buttons/section titles · 26 summary name · 32 panel titles · 64 logo |
| Minimum sizes | primary button 0×48, nav back 100×40, nav next 140×40, select spell 130×38, input 0×48 (auth) / 0×44, avatar panel 100×100, race card 200 wide, spell slot 80×80, icons 52 (detail) / 48 (browse, summary), summary avatar 80 |
| Touch targets | ≥ 48 design units (tutorial and HUD rule) |

Auth card specifics: inputs 78 px tall, radius 12, border `(1,1,1,0.11)`, left margin 88 for a 34 px icon; card `(0.035,0.03,0.03,0.9)`, radius 18, orange shadow glow (`shadow_size 30`, offset `(4,10)`).

## Patterns

Menu click revision (HEX-15, 2026-09-10): MenuPlayer now synthesizes a 110 ms filtered noise/wood transient with a subdued low body and soft attack/tail, replacing the 1250 Hz sine beep. The shared button wiring remains; master volume applies.

- Card states: default no border; selected/focused accent 3 px; chosen (spell browse) green 2 px; same background.
- Validation: error labels hidden by default, red, centered; points/count labels yellow while incomplete, green when satisfied; sliders clamp instead of erroring.
- Screen skeleton: `Control` → `BgTexture` (cover) → `BgOverlay` → `Main` `MarginContainer(60/30/60/20)` → `Layout` `VBoxContainer(16)` → header/step bar + `Content` panel → step panels toggled by visibility.

## Mobile layout revision (2026-09-08)

Compaction uses the larger pressure from **height 1000→612** and **width 2200→1360**, clamped to 0–1. It handles Label and RichText font overrides and preserves readable fonts (floor 15) and touch targets; clickable primary cards use the same touch floor as buttons. Screen outer margins respect Android/iOS display safe-area insets even when compaction is unnecessary. The background still covers the full viewport.

- Creation: compact mode hides the redundant side summary. Race selection shows name, tagline, lore and active racial traits; no racial min/max or starter-stat bonus sections. Selecting a card preserves its compact padding. Dragging the race list does not select a race. The step content scrolls and Back/Next stay outside that scroll. Decorative connectors collapse below 2000 wide; step captions below 1400 (the current-step caption remains).
- Character detail: all compact tabs can scroll. Below 1700 wide the tab strip has its own row, summary sections and primary cards stack. Stats/spellbook stack below 1250; the entire summary stacks below 900 or when its measured columns cannot fit. Column arrangements are recalculated on viewport resize, including widening again. Selection styles retain fitted padding.
- Dashboard: flexible card widths and a smaller PLAY minimum; content scrolls independently of the bottom navigation. Cached node references keep asynchronous character refresh working after reparenting. Match mode overlay uses compact typography and spacing.
- Settings: the three settings columns stack below 1200; the footer remains outside the content scroll. Lobby below 550 high drops header decoration and scrolls its content area. Results below 612 high scroll as a whole, preserving access to Continue; captions have a 15-unit font floor, action icons are at least 48×48, and long level-up text wraps.

Verification harness: `Game/ScenesV3/Dev/MobileLayoutVerification.tscn`; `MOBILE_SIZE=1360x612`, optional `MOBILE_CAPTURE=/absolute/output` for rendered PNGs, `MOBILE_RESIZE=1` to exercise detail-tab resizing. It uses offline fixtures, no account mutation. See [[2026-09-08-mobile-layout-review]] for coverage and limitations.

## Arcane scrollbars (2026-09-08)

`ArcaneScrollGlow.ApplyTo` skins native vertical/horizontal ScrollContainer bars and active RichTextLabel bars. ResponsiveLayout applies it in compact and desktop layouts, including newly wrapped content. Dashboard, News, Social and Lobby no longer hide their scrollbar or disable its input. Fit-content RichText content directly inside a ScrollContainer uses the outer scroll.

- Touch lane: 48 design units; visible rounded bronze/gold thumb: 24; dark bronze-bordered track: 8. Minimum thumb length: 48 even for very long lists. Hover/pressed/focus states remain visible.
- A central etched diamond rune, a subtle 4.5-second amber breathing glow and a 5.5-second traveling glint make the handle discoverable. Decoration ignores input, redraws at most 30 times per second and stops processing while hidden. Animation never changes scroll position or layout.
- Native Godot mouse, touch and keyboard scrolling are retained. The transparent gutter around the visible thumb accepts dragging.
- Below 1100 wide, creation summary columns and Social content stack; Social content scrolls. Lobby uses two spell columns and a smaller book minimum to accommodate the wider scroll lane.

Verification: `Dev/ScrollbarVerification.tscn` checks visibility, 48-unit target, native mouse and synthetic touch drag (including the gutter), and minimum grabber size with long content. MobileLayoutVerification optionally records the animation with `SCROLL_ANIMATION_CAPTURE=/absolute/output` and asserts the scroll value stays fixed. Rendered screenshots, animation GIF and logs: `verification/arcane-scrollbar/` in the client workspace. Physical Android touch/DPI remains a manual follow-up.

## Summary touch scrolling (HEX-7, 2026-09-10)

CharacterDetail's Summary passes pointer input from its portrait, card panels, generated icons/text and learned-spell rows through to the enclosing native ScrollContainer. A drag moves the whole Summary content with Godot's existing inertia and drag threshold; a tap on a learned-spell row opens Spellbook with that spell selected (HEX-8). Header navigation remains outside the content scroll, and the scrollbar remains available as a position indicator and direct control.

`EnableSummaryTouchScroll` changes only `MouseFilter.Stop` to `Pass`, preserving existing Ignore controls. It runs after wrapping/reflow and `CompactRefresh`, including asynchronous creation of learned-spell rows. Other tabs and shared control factories are unchanged. Desktop mouse-wheel input continues to work over the content.

Verification scene: `Dev/SummaryScrollVerification.tscn`, optional `MOBILE_SIZE=1360x612` and `SUMMARY_CAPTURE=/absolute/file.png`. It exercises the real screen with generated spell data and native emulated-touch input over portrait, stat/spell/skill/modifier icons and labels, spell-row drag, spell-row tap and desktop wheel. All 12 interaction checks pass at 1360×612, 1088×612 and 844×390, and in a rendered 1360×612 run. Build: zero errors / 11 existing warnings. Logs and screenshot: client `verification/summary-scroll/`. Physical Android touch remains a follow-up. An exploratory 390×844 portrait fixture exposed existing horizontal clipping (Summary minimum width exceeds the viewport); that layout issue is outside this gesture fix, and offscreen synthetic touches are not counted as coverage.

Source: `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.Responsive.cs`, `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.cs`, `client:Game/ScenesV3/Dev/SummaryScrollVerification.cs`.

## Summary organization and spell shortcuts (HEX-8, 2026-09-10)

Summary has two columns. The left column contains a naturally sized character/stat card, compact Attributes rows, then Modifiers. The right column contains Skills followed by the complete learned-spell list. The character card no longer stretches to the height of the entire page; its portrait/stat rows are more compact. Modifier names wrap to fit the narrower card. Columns retain the responsive stacking behavior.

All learned spells, including learned standard spells, are shown; the old five-row cap and View all spells button are removed. Each spell row supports pointer tap and keyboard `ui_accept`, with hover/focus highlighting. Touch movement at or above the existing tap threshold cancels activation; descendants ignore pointer events so the row and native Summary scroll receive them. Summary now has a ScrollContainer on desktop too, since the complete learned list can exceed the viewport. Its visibility follows the active tab, including direct Dashboard shortcuts to Stats/Spellbook/Primary.

A row opens the existing Spellbook tab, selects the matching spell, updates its details/highlight, and reveals the selected grid card plus name/description in the outer scroll. It reuses the current catalogue and requires no new RPC. Delayed scrolling aborts if the screen is removed or the selected spell/tab changes. Standard spells still offer their existing Primary-development action within Spellbook.

Verification: `Dev/SummaryReorganizationVerification.tscn` checks eight learned spells plus an excluded unlearned spell, card order, absent old button, viewport fit, per-spell selection/details visibility and desktop tab shortcuts. `Dev/SummaryScrollVerification.tscn` now tests tapping/dragging learned rows instead of the removed button. Validation: 37 checks pass at 1360×612 and 844×390; 40 pass at desktop 2400×1080 including Dashboard shortcuts. The gesture regression passes 12 checks on both landscape phone sizes. Build: zero errors / 11 existing warnings. Rendered screenshots and logs: client `verification/summary-reorganization/`. Physical Android remains a follow-up; no client installation or backend changes are part of HEX-8.

Source: `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.tscn`, `CharacterDetailScreen.cs`, `CharacterDetailScreen.Responsive.cs`.

## Source of truth in code
- `client:Game/ScenesV3/_Themes/m_charcreate_theme.tres`, `m_auth_card_theme.tres`, `m_auth_theme.tres` — fonts, variations, styleboxes
- `client:Game/ScenesV3/_Common/ResponsiveLayout.cs`, `UiKit.cs`, `ArcaneScrollGlow.cs` — compact layout, shared builders and animated native scrollbar skin
- `client:Game/ScenesV3/GameOver/GameOverScreen.cs` — Cinzel Bold/Black display fonts
- `client:Resources/Fonts/*/static/` — the shipped static faces

## Combat desktop/mobile profiles (2026-09-09)

See [[combat-ui-profiles]] for the shared profile mechanism, configurable keyboard actions, preview scenes and validation. Desktop uses the bottom action bar and visible keycaps; mobile keeps touch rails. The tutorial follows the chosen profile and respects configured keys and target gates.

Pre-match arrangement now shares `ReferenceSpellSlot` and `CombatSlotGeometry` with combat. It uses the existing screen background and a scrollable spell detail card, without character HUD. See [[combat-ui-profiles#Pre-match spell arrangement (2026-09-09)]].


## Illustrated duel loading (2026-09-10)

See [[duel-loading-screen]] for the persistent blue/emerald loading illustration, threaded resource progress, one stable random tip, responsive mobile layout and retry/cancellation verification. This user-requested background is a deliberate exception to shared menu artwork. It replaces only the arrangement-to-arena black transition.

## Primary, lobby feedback and menu affordances (HEX-9, HEX-10, HEX-11, HEX-15, 2026-09-10)

The Primary tab presents one path at a time through an explicit Magic Arrow / Mirror Reflection switch. Each progression tier is a compact left-to-right column of icon buttons; the icon tooltip and inline selection line expose the node title and effect, while selecting a reachable icon still updates the pending path.

Lobby draft slots retain selected spell data for both players. Filled slots show the spell name as a tooltip and are clickable; selecting either your own or the opponent's spell opens the same icon, metadata and description panel used by the spell library. Dashboard stat points, magic points and pending Primary tiers use a slow amber pulse on their existing cards while work is available. MenuPlayer generates a short PCM click at runtime and binds it to every `BaseButton` in the current scene, including dynamic plus/minus controls.

Sources: `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.Primary.cs`, `client:Game/ScenesV3/Lobby/LobbyScreen.cs`, `client:Game/ScenesV3/Dashboard/DashboardScreen.cs`, `client:Game/Autoloads/MenuPlayer.cs`.
