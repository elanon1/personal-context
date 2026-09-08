---
type: project
project: Hexbane
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
tags: [hexbane, docs, index, moc]
aliases: [hexbane-docs, hexbane-index]
---

# Hexbane — mapa dokumentacji (jedyne źródło prawdy)

Ten folder (`10_Projects/hexbane/docs/`) jest **jedyną** dokumentacją projektu Hexbane. Repozytoria
`elanon1/hexbane` (klient Godot/C#) i `elanon1/hexbane-server` (Go/Nakama) nie mają własnych docs —
ich `docs/README.md`, `CLAUDE.md` i `AGENTS.md` odsyłają tutaj.

## Reguły (dla ludzi i agentów: Claude Code, Codex)

1. **READ first** — przed zmianą protokołu/opcode’ów, RPC, zaklęć, ras, progresji, auth, deployu,
   infry lub UI przeczytaj właściwą notatkę poniżej. Notatka jest kontraktem; jeśli kod się z nią nie
   zgadza, zweryfikuj w kodzie i popraw notatkę.
2. **WRITE in the same task** — zmiana zachowania = aktualizacja notatki w tym samym zadaniu
   (podbij `updated:` i `verified:` we frontmatter). Brak pasującej notatki → nowa w odpowiednim
   `docs/<obszar>/` + wpis tutaj.
3. **LOG every session** — wpis w [[dziennik]] (data, co zrobione, notatki/pliki, co zostało).
   Decyzje z uzasadnieniem → [[10_Projects/hexbane/_state|_state]] → *Decisions log*.
4. **NEVER** twórz dokumentacji w repo (`docs/*.md`, `RPCs.md`, `GUIDE*.md`, `thoughts/`, specs/plans
   superpowers). Plany i specyfikacje → `docs/plans/`.
5. Brak vaulta na maszynie → zatrzymaj się i powiedz użytkownikowi; nie pisz docs do repo.
6. Zwykłe narzędzia plikowe na ścieżce vaulta są kontraktem; MCP Obsidian to opcjonalna wygoda.

Konwencje notatek: frontmatter `type: project, project: Hexbane, area, status, created, updated,
verified, tags, sources`; treść techniczna po angielsku; każda notatka kończy się sekcją
**Source of truth in code** z ścieżkami `client:` / `server:`. Wikilinki po nazwie pliku.

## Stan projektu

- [[10_Projects/hexbane/_state|_state]] — podsumowanie, status, **decisions log**, open questions
- [[dziennik]] — dziennik pracy (sesje, zmiany, decyzje)
- [[repos-and-branches]] — snapshot obu repo (branch, niezacommitowana praca, ostatnie commity)

## Protokół klient ↔ serwer (`docs/protocol/`)

- [[opcodes]] — indeks wszystkich opcode’ów (live + wycofane), numery, kierunki, fazy
- [[combat-v2]] — kontrakt walki `duel_v2` (protokół 2, tick 100 ms, 29 → 30/31/32, reconnect)
- [[shared-types]] — wspólne typy payloadów
- [[rpcs]] — pełny rejestr RPC (request/response, błędy, kto woła)
- [[character-details]] — kontrakt `get_character_details` / ekran postaci
- [[race-selection]] — wybór rasy i tworzenie postaci (3/4 startery)
- [[spell-visual-key]] — `visual_key` (klient gotowy, backend nie wysyła)
- Opcode’y: [[op_00_match_entry_data]] · [[op_01_server_ready]] · [[op_02_client_ready]] ·
  [[op_03_lobby_update]] · [[op_04_lobby_spell_selected]] · [[op_05_lobby_spell_selected_update]] ·
  [[op_06_launch_game]] · [[op_07_game_ready]] · [[op_08_match_declined]] · [[op_09_match_canceled]] ·
  [[op_10_game_data]] · [[op_16_game_countdown]] · [[op_29_combat_command]] · [[op_30_combat_result]] ·
  [[op_31_combat_event]] · [[op_32_combat_snapshot]] · [[op_50_game_over]] ·
  [[op_70_lobby_spellbook_spells]] · [[op_100_story_update]] · [[op_101_story_choice_selected]] ·
  [[op_199_quit_game]]

## Serwer (`docs/server/`)

- [[server-architecture]] — moduły, silnik meczu, maszyna faz z czasami, stan, współbieżność, bot
- [[dev-setup]] — make/docker compose, env, migracje, testy, symulator
- [[matchmaking]] — `normal` vs `ai_duel`, matchmaker, rejoin
- [[spell-system]] — katalog 14 zaklęć, schema YAML, efekty, kolejka, jak dodać zaklęcie
- [[spell-lore]] — natury i inkantacje, `get_spell_lore`
- [[combat-stat-rules]] — aktywne statystyki, odporności, regeneracja, skille i bonusy ras w duel_v2.3
- [[progression]] — 6 ras, staty, skille, XP/poziomy/MP, sloty, reguły walki duel_v2
- [[database]] — schemat z migracji `000001..000003`, tabele Nakamy, reset
- [[social]] · [[notifications]] — moduły na wbudowanych API Nakamy
- [[server-tutorial]] — RPC `tutorial` i flagi
- [[google-auth]] — kontrakt logowania Google (wbudowane `AuthenticateGoogle`)

## Klient (`docs/client/`)

- [[client-architecture]] — warstwy, autoloady, DI, CQRS, przepływ wiadomości, trasy scen, testy
- [[duel-v2-client]] — implementacja pojedynku (scena, HUD, klawisze, snapshoty, reconnect, weryfikacja)
- [[cqrs]] — jak używać dispatchera / handlerów
- [[spell-effect-system]] — architektura efektów zaklęć (interfejsy, rejestr, konfiguracje)
- [[spell-vfx-configuration]] — które z 14 id mają VFX/SFX/ikony, presety `SpellVisuals`, jak dodać
- [[vfx-and-race-animation]] — sprite’y ras, `RaceSpriteAnimator`, gesty, cast-charge, okluzja,
  medytacja, bariera, areny; sceny dev i weryfikatory (`verification/` w repo)
- [[design-system]] — tokeny UI, fonty, motywy (`ui-designer` jest do niego przywiązany)
- [[social-sign-in]] — Google sign-in na PC i Androidzie, konfiguracja, znane luki
- [[client-tutorial]] — lokalny tutorial
- [[deploy-android]] — `deploy.sh`, ADB, LAN host, presety eksportu
- [[legacy-and-tooling]] — martwe trasy/kod, `.env`, stary pipeline n8n, archiwa sztuki, addony

## Infra i assety (`docs/infra/`)

- [[infra-and-deploy]] — lokalny stack, obraz, CI, Helm/Argo (nieużywane), polityka „tylko lokal”
- [[assets-pipeline]] — jak powstaje sztuka (rasy Tripo→Mixamo→Blender→sheety, areny, ikony)
- [[repos-and-branches]] — snapshot repo 2026-09-07

## Plany (`docs/plans/`)

- [[2026-09-08-xp-ranked-progression]] — szybki rozwój do rankedów, łagodnie rosnące koszty poziomów i dalszy rozwój skilli/MP bez blokady wejścia.

- [[2026-09-08-race-primary-progression-redesign]] — propozycja redesignu ras, wspólnego budżetu statów i grafów rozwoju primary spelli (6 poziomów, checkpointy3/6).

- [[2026-09-08-fallback-player-design]] — projekt naturalnego przeciwnika zastępczego: kolejka, persony, lobby i AI.
- [[2026-09-08-fallback-player-plan]] — plan wdrożenia i weryfikacji fallbacku (propozycja, bez implementacji).

- [[2026-09-08-combat-restoration]] — przywrócenie mechanik walki po zatwierdzeniu użytkownika

- [[2026-09-08-client-legacy-cleanup]] — usunięcie starych VFX/SFX, storytellingu i martwych zależności klienta.

- [[2026-09-05-spell-system-redesign]] — co z redesignu zostało do zrobienia
- [[level-up-notifications]] — niezaimplementowany plan powiadomień o level-upie
- [[server-devlog-summary]] — oś czasu kamieni milowych serwera (z dawnego `devlog.md`)

## Audyty (`docs/audits/`)

- [[2026-09-08-balance-review]] — rewizja ras, statcapów, skilli i integracji czarów; 10 860 symulowanych walk, problemy i propozycje korekt.

- [[2026-09-08-server-unused-code-audit]] — audyt martwego kodu serwera: 61 kandydatów, zależności i kolejność czyszczenia.

Raporty weryfikacji docs vs kod z 2026-09-07 — sekcje „code smells” i „open questions” to backlog:
[[2026-09-07-server-match-audit]] · [[2026-09-07-server-spells-audit]] ·
[[2026-09-07-client-docs-audit]] · [[2026-09-07-readmes-audit]] · [[2026-09-07-vault-infra-audit]] ·
[[2026-09-07-vault-state-audit]] · [[2026-09-07-vault-notes-audit]]

## Archiwa

- `99_Archive/Projects/hexbane-notes-2026-08-31/` — poprzednie polskie notatki vaulta (sprzed duel_v2)
- `~/hexbane-archive/docs-2026-09-07/` — usunięte z repo drzewa `docs/`, `thoughts/`, `RPCs.md`
- `~/hexbane-archive/Races-2026-08-31/` — wycofany roster 35 ras

- [[docs/plans/2026-09-08-redesign-implementation|Wykonanie przebudowy ras, primary i progresji]] — zakres, walidacja, migracja i ograniczenia.
