---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
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

## Source of truth in code
- `client:Game/ScenesV3/_Themes/m_charcreate_theme.tres`, `m_auth_card_theme.tres`, `m_auth_theme.tres` — fonts, variations, styleboxes
- `client:Game/ScenesV3/_Common/ResponsiveLayout.cs`, `UiKit.cs`, `ArcaneScrollGlow.cs` — compact layout, shared builders and animated native scrollbar skin
- `client:Game/ScenesV3/GameOver/GameOverScreen.cs` — Cinzel Bold/Black display fonts
- `client:Resources/Fonts/*/static/` — the shipped static faces
