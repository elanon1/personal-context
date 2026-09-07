---
type: project
project: Hexbane
domain: [projects, creative]
status: archived
created: 2026-08-31
updated: 2026-09-07
archived: 2026-09-07
original-path: 10_Projects/hexbane/assety-i-pipeline.md
superseded-by: "[[_index]] (10_Projects/hexbane/docs/)"
tags: [hexbane, ai-art, assets, races, spells, pipeline, chatgpt, adobe]
aliases: [hexbane-assets, hexbane-races]
---

> [!warning] Zarchiwizowane 2026-09-07 — opis sprzed przeprojektowania duel_v2. Aktualna dokumentacja: `10_Projects/hexbane/docs/` (patrz `[[_index]]`). Audyt rozbieżności: `[[2026-09-07-vault-notes-audit]]`.

# Hexbane — assety i pipeline AI-art

Stan na 2026-08-31. Część [[10_Projects/hexbane/_state|Hexbane]]. Operacyjne „jak generować” żyje w skillach repo (`.claude/skills/`) i w pamięci Claude Code projektu; tu — co istnieje i w jakim stanie.

## Rasy (`Resources/Races/`)

- **Roster pusty od 2026-08-31** — 35 ras (umbrakin, vitrael, drosskin, … amberlith) wycofane, nowe będą od zera. Stara sztuka **w archiwum** `~/hexbane-archive/Races-2026-08-31/` (922 MB: 35 folderów ras + `_raw_chroma/` + `_style_comparison/` + `_server-docs-client-races/` ze starymi biblami 8 ras i tumbloama). Nigdy nie było tego w gicie. W repo zostało `Resources/Races/_tools/{key_sheet.py, build_race_frames.py}`.
- **Czego oczekuje kod od rasy** (to zostaje): folder `Resources/Races/<race_id>/` z `avatar.png`, `front.png` (+ opcjonalnie `portrait_neutral.png`, `full_body_neutral.png`) — ścieżki w `Core/Characters/Character.cs`; `animation/frames.tres` (`SpriteFrames` z animacją `idle`) dla podglądu w tworzeniu postaci (`RaceAnimationPreview`); na serwerze wiersz w `races` (migracja) z `race_id` = nazwa folderu. Lista ras w UI pochodzi z RPC `get_races`, więc bez migracji rasa nie istnieje.
- **Format kompletnej rasy wg skilla `race-maker`**: `character_bible.md` + `chat_gpt_reference.md` + stille `idle/avatar/front` + `animation/{idle, spell_throw, meditation, victory, dead, damage_taken}/<nazwa>_sheet.png` + `images/frame_0..8.png` (3×3). Pipeline: tło = jednolita chroma per rasa (zielona `#00FF00` domyślnie, magenta/niebieska gdy paleta koliduje), **zero malowanych efektów/glow** (VFX dodaje silnik), figura ~60–65 % komórki, ~30 px marginesu, klatka 1 = Idle; jeden wątek ChatGPT na rasę, idle jako referencja. Skill jest w pełni autonomiczny (nie pyta). Skrypty: `.claude/skills/race-maker/scripts/{extract_grid.py, slice_sheet_alpha.py, upscale_frames.jsx, ps_cutout.jsx}`.
- **W pojedynku rasy nie są renderowane** — `Player.tscn` to generyczny rig z `Resources/Chars/Concept1`. Eksperymentalne sceny rigu (`RaceTest/`) i podgląd `Dev/VitraelPreview` usunięte razem ze starymi rasami (w historii gita, HEAD `01ec454`).

## Zaklęcia (`Resources/Spells/`)

- 19 folderów z ikoną `<id>.png`: aqua_pulse, arcane_shield, cure, ember_burst, explosion, fireball, flamestrike, frost_cut, great_heal, gust, heal, magic_sparkle, mirror_ward, paralyze, poison_dart, reflection, spark, stoneguard, venom_shot.
- 10 z pełnym pipeline'em (`spell.json` + `manifest.generated.json` + `primitives/textures/motions/references`, linki Google Drive z n8n): arcane_shield, cure, explosion, fireball, flamestrike, heal, magic_sparkle, paralyze, poison_dart, reflection — czyli **stary prototyp**, nie obecny katalog 63 zaklęć serwera.
- Pokrycie: z 63 zaklęć serwera tylko ~9 ma jakąkolwiek ikonę w kliencie, VFX aktywne dla 9 konfiguracji (3 z nich to id nieistniejące na serwerze). `Resources/Icons/` — 31 legacy SVG. Audio: `fireball.wav`, `heal.*` — prawie nic.
- Prompty n8n: `prompt.md` (zaklęcie → JSON z promptem ikony i audio), `vfx.md` (definicja → `vfxBlueprint`; woła `get_spell_details_yaml` na Nakamie), `spell_output.md` (schemat `spell.json`). Komendy `.claude/commands/create_spell_{icon,vfx,textures,sfx,sounds}.md`.

## Środowiska / areny

Skille `env-concept` (bible + master concept) i `env-maker` (5 warstw parallax + FX + `environment.json`) — `Resources/Environments/` istnieje; aktywna scena meczu używa `Resources/Backgrounds/Sky/` + TileMapLayer.

## Narzędzia w repo klienta

- Addony: `godot_mcp` (Godot MCP Pro 1.14.1, WebSocket :6505, serwer Node w `mcp_server/`), `ColorPreview`, `rider-plugin`. Gotchas MCP (scena aktywna vs `create_scene`, tekstury przez `execute_editor_script`, tracki animacji) — w pamięci Claude Code.
- `remove_bg.py` (rembg U2Net), `Scripts/` (uv Python).
- `deploy.sh` → Android APK przez Godot CLI na Windows + ADB po WiFi (ścieżki `F:/`, `/mnt/c/` — WSL2, nie Mac).
- `.claude/`: 7 agentów (m.in. `ui-designer`), 30 komend, 3 skille; `MAX_THINKING_TOKENS=32000`.
