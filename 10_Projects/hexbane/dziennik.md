---
type: project
project: Hexbane
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, log, worklog]
aliases: [hexbane-dziennik, hexbane-worklog]
---

# Hexbane — dziennik pracy

Najnowsze wpisy na górze. Każda sesja (człowiek, Claude Code, Codex) dopisuje wpis: **data — agent —
co zrobione — notatki/pliki — co zostało**. Decyzje z uzasadnieniem idą dodatkowo do
[[10_Projects/hexbane/_state|_state]] → *Decisions log*. Historia sprzed dziennika (2025-06 →
2026-09-06): [[server-devlog-summary]] i decisions log w `_state`.

## 2026-09-08 — Codex — czyszczenie helperów i naprawa progresji

**Zrobione** — zgodnie z decyzją użytkownika usunięto niepodłączone helpery/walidatory i funkcje debugowe, sześć helperów powiadomień z martwymi DTO oraz stary bufor `CastInterruptions`. `MatchLog` zachowany. `get_progression` korzysta ze wspólnych funkcji XP/slotów, test regresji sprawdzono przed i po naprawie. Testy paraliżu sprawdzają obecne zdarzenia lifecycle, brak zwrotu many i wpisy MatchLog. Zachowano kod/testy odporności, regeneracji i skilli do przywrócenia. Usunięto 33 deklaracje funkcji/metod (w tym zastąpione `pow`). Końcowe `go test ./...`, `go vet ./...` i `git diff --check`: PASS; testy race match/spell_system/symulator: PASS po zmianie bufora.

**Pliki/notatki** — moduły character, spellbook, spell_system, notifications, social, race, common, match oraz lokalne nieużywane helpery endless_story; [[rpcs]], [[progression]], [[spell-system]], [[server-architecture]], [[notifications]], [[2026-09-08-server-unused-code-audit]], [[_state]], ten dziennik. Nie zmieniano migracji, RPC rejestracji ani katalogu zaklęć. Bez commita.

**Zostało** — doprecyzowanie zakresu powrotu mechanik walki i ich aktywacja; obecnie nadal działa model stałych wartości duel_v2. Nie wycofywano story ani działających RPC powiadomień.

## 2026-09-08 — Codex — audyt nieużywanego kodu serwera

**Zrobione** — przeanalizowano aktualny working tree (niezacommitowane zmiany zaakceptowane przez użytkownika), 451 deklaracji funkcji/metod. Wskazano 61 kandydatów bez odwołań produkcyjnych po wykluczeniu callbacków: 48 bez testów, 13 z odwołaniami testowymi. Zweryfikowano martwe formuły walki, zależności skilli, bufory zdarzeń, helpery powiadomień, niedokończone story i duplikację progresji. Przygotowano kolejność czyszczenia oraz granice API/danych. `go test ./...` (cache) i `go vet ./...`: PASS, Go 1.24.5 na macOS. Analiza AST i ręczna; nie pełny graf osiągalności typowanego programu.

**Pliki/notatki** — [[2026-09-08-server-unused-code-audit]], [[_index]], ten dziennik. Kod aplikacji pozostawiono bez zmian; nic nie zacommitowano.

**Zostało** — realizacja opisanych etapów czyszczenia, naprawa starej drabinki slotów w `get_progression`, decyzja o wycofaniu story i ewentualnym zmniejszeniu API. Raport to rekomendacje, nie zatwierdzone decyzje produktowe; przed usuwaniem RPC potrzebny aktualny audyt klienta.

## 2026-09-07 — Claude Code — konsolidacja dokumentacji

**Zrobione**
- Zweryfikowano z kodem (working tree obu repo) i scalono dwa drzewa docs: klient `docs/` (45 plików
  md + kopie docs serwera + 804 MB zrzutów) i serwer `docs/` + `RPCs.md` (75 plików). Wynik:
  `docs/{protocol,server,client,infra,plans,audits}` — patrz [[_index]].
- Usunięte jako nieaktualne/duplikaty (kopia w `~/hexbane-archive/docs-2026-09-07/`): serwer
  `docs/match/*` (GUIDE-v2, README, architecture, phases, state-management, communication,
  matchmaking, api-reference), `QUICKSTART-v2`, `API-REFERENCE-v2`, `DOCUMENTATION-INDEX`, stuby
  `spell_system/GUIDE-v2`, `client/effect-sync`, tombstone’y opcode’ów 11–15/21–28, specs/plans
  superpowers (zaimplementowane), `devlog.md` (streszczony); klient `docs/Server/*` (stare
  progression: 35 ras, sloty 4–10, formuły ze statów), `docs/opcodes/*` (stare payloady), `Plans/`,
  `superpowers/`, `thoughts/`, README’y w `Application/CQRS`, `Application/ArcaneDuel`, `Game/`,
  `Resources/SpellVisuals`, pusty `Scripts/README.md`.
- Repo: sztywna reguła „docs tylko w vaultcie” w `CLAUDE.md` + `AGENTS.md` obu repo;
  `docs/README.md` jako jedyny plik wskazujący tutaj; `.claude/commands/opcodes.md`,
  `.claude/agents/ui-designer.md` (klient) i `.claude/agents/opcode-docs.md` (serwer) przepięte na
  vault; zrzuty weryfikacyjne → `verification/` (`.gdignore`, gitignore) i 7 skryptów `Verify*.cs`
  zaktualizowanych; komentarze XML w kodzie odsyłające do starych ścieżek poprawione.
- Vault: `_state.md` przepisany pod stan duel_v2; stare notatki → `99_Archive/Projects/hexbane-notes-2026-08-31/`;
  `NOW.md` zaktualizowany; raporty rozbieżności w `docs/audits/`.
- Codex: `~/.codex/config.toml` wymaga ręcznego dopisania `[sandbox_workspace_write]
  writable_roots = ["/Users/elanon/PersonalContextEngine/Personal Context Engine"]` (edycja
  zablokowana przez klasyfikator uprawnień Claude Code).

**Najważniejsze znaleziska** (szczegóły w audytach)
- Serwer: `MatchLog` pisany i wyrzucany co tick; opcode 2 ufa `user_id` z payloadu; opcode 10 wysyła
  zaklęcia przeciwnika; `create_character_match_story` woła nieistniejący mecz; 7 z 8 traitów ras bez
  konsumenta; `get_progression` liczy XP/sloty starą drabinką; `balance-v2.json` = `duel_v2.1` przy
  kodzie `duel_v2.2`; XP/MP w starych docs różniły się od kodu o 1 poziom.
- Klient: 15 autoloadów (CLAUDE.md wymieniał 6); rejestr VFX kluczowany 20 starymi id FX, 14 id
  serwera mapowane aliasami; stare foldery handlerów 11–15/21–28 wciąż kompilowane; martwe trasy
  `SceneManager`; `docs/client/reference-duel` bez `.gdignore` importowało 53 MB PNG do `.godot/`.

- Usunięte komendy `.claude/commands/create_spell_{icon,sfx,sounds,textures,vfx}.md` (stary pipeline generowania zaklęć; `old_create_spell.md` zostawiony).

**Zostało / do decyzji** — lista *Open questions* w [[10_Projects/hexbane/_state|_state]]; nic nie
zacommitowane (ani repo, ani vault).


## 2026-09-08 — Magic Arrow: rozpoznanie i kierunek VFX

- Prośba: efekt samego zaklęcia o jakości docelowej AAA, bez animacji castingu.
- Przeczytano indeks i dokumentację klienta: spell-effect-system, spell-vfx-configuration, vfx-and-race-animation; sprawdzono MagicSparkle.cs i mapowanie magic_arrow → magic_sparkle.
- Przygotowano propozycję: ostry świetlisty grot, warstwowa smuga energii, kierunkowe trafienie z krótkim błyskiem i wygasającymi odłamkami.
- Zmieniono wyłącznie dziennik; kod i assety bez zmian. Pozostało zatwierdzenie kierunku wymagane przez skill brainstorming, implementacja, build i ocena efektu w Godot.


## 2026-09-08 — Czyszczenie starych czarów, VFX/SFX i storytellingu klienta

- Usunięto 863 pliki (287,5 MiB): 19 starych implementacji VFX wraz ze scenami/teksturami/particle assets, dane generatora zaklęć, wszystkie SFX (w tym Mirror Reflection i klik UI), klientowy storytelling (RPC, manager, DTO, handlery/komendy, DI, event, opcode’y 100/101), nieużywane handlery starej walki i GTweens.
- Zachowano nowe animacje ras, casting/gesty/charge/occlusion/presety, medytację, areny, nowy MirrorWard i 13 ikon obsługujących 14 aktualnych zaklęć. Muzyka została. Stare id w ścieżkach ikon są aliasami assetów, nie aktywnymi starymi zaklęciami.
- Uproszczono registry/factory/manager do efektów statycznych i MirrorWard; usunięto nieaktywne interfejsy projectile/beam/area, martwy cleanup, stare nawigacje i test Heal. Podglądy lobby/HUD korzystają z obecnego katalogu.
- Walidacja: build 0 błędów / 9 ostrzeżeń (przed zmianami 37); MirrorWard 60/60; GestureVfx 8899/8899; aktywny HUD duel_v2 i 14 ikon PASS; brak odwołań zasobów do usuniętych plików. Test gestów dostosowany do istniejącego zachowania presetów (brak podwójnego generic charge), bez zmiany castingu.
- Offline test aktywnego HUD-u loguje null-reference inicjalizacji NotificationModule oraz ObjectDB leaks przy wyjściu; nie naprawiano niezwiązanych modułów. Nie sprawdzano meczu na żywym Nakama ani Androida.
- Notatki: spell-effect-system, spell-vfx-configuration, vfx-and-race-animation, client-architecture, legacy-and-tooling, opcodes, op_100_story_update, op_101_story_choice_selected, rpcs, _index, _state, docs/plans/2026-09-08-client-legacy-cleanup.
- Kopie usuniętych plików (również untracked) w /tmp/hexbane-removed-{legacy,code,pipeline}.tar.gz; manifest /tmp/hexbane-removed-files.json. Nie robiono commitów. Zakres: klient; backend storytellingu nie był modyfikowany.


## 2026-09-08 — Przywrócenie dźwięków Mirror Reflection

- Na prośbę użytkownika przywrócono oryginalne formation.wav i shatter.wav (identyczne z kopią sprzed czyszczenia), WardAudio.cs oraz obsługę audio w MirrorWard.cs. Przywrócono VerifyWardAudio.cs/.tscn i metadane UID/import.
- Pozostałe stare SFX nie zostały przywrócone.
- Walidacja: dotnet build — 0 błędów, 9 ostrzeżeń; Godot VerifyWardAudio headless — 10 kontroli, 0 błędów (start, bus, brak podwójnego shatter, wygaszanie, ogon dźwięku i cleanup).
- Zaktualizowano spell-vfx-configuration, spell-effect-system, vfx-and-race-animation, client-architecture, legacy-and-tooling i _state. Nic nie zostało do wykonania w tym zakresie; bez commita.

## 2026-09-08 — Codex — przywrócenie mechanik serwera

Przywrócono profil STR/INT/DEX i ras, odporności, unik, regenerację HP/many, medytację, skalowanie obrażeń/leczenia i wzrost/zapis skilli. Poprawiono zaokrąglanie terminów ticków, ułamkową manę przy pełnym zasobie i efektywne metadane czarów w drafcie/prywatnych widokach/karcie postaci. MatchLog zachowany, stary bufor i wyczyszczone helpery nie wróciły; get_progression pozostaje naprawiony.

Pliki: `modules/combat`, `modules/character/details*`, `modules/match`, `modules/skills/gain.go`, `modules/spell_system`, `cmd/duel-sim`, `scripts/test_combat_runtime.mjs`; klient `Core/Match/DuelV2.cs` — zaakceptowana kompatybilność katalogu2.3. Notatki: combat-stat-rules, progression, spell-system, server-architecture, combat-v2, rpcs, character-details, shared-types, opcode31/70, duel-v2-client, plan, audit, _index, _state.

Weryfikacja: `go test ./...`, `go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim`, `go vet ./...`, `make build` — PASS. Klient `dotnet build hexbane.csproj --no-restore`: 0 błędów, 9 wcześniejszych ostrzeżeń. Symulator: 100 meczów, seed42. Review: naprawione dwie uwagi z testami regresji.

Pozostało poza zakresem: test live po restarcie lokalnego stacku, tuning balansu i ewentualna jawna klasyfikacja szkół/typów obrażeń w katalogu. Nie wdrażano zmian ani nie commitowano.

## 2026-09-08 — Codex — instrukcja przebudowania

Sprawdzono Makefile, docker-compose.yml i docs/server/dev-setup. Po zmianach Go lokalny cykl to `make dev` (build pluginu, kopia do kontenera, restart Nakama); klient wymaga przebudowania po zmianie kompatybilności katalogu. Bez uruchamiania stacku i bez zmian kodu.
