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


## 2026-09-08 — Codex — projekt naturalnego przeciwnika zastępczego

- Na prośbę użytkownika opracowano projekt i plan, bez implementacji backendu/klienta. Przeczytano kontrakty vaulta, bieżący matchmaking, boty, lobby, walkę, progresję i klientowy MatchManager; uwzględniono niezacommitowane przywrócenie statystyk/skilli.
- Pliki: docs/plans/2026-09-08-fallback-player-design.md, docs/plans/2026-09-08-fallback-player-plan.md, _index.md, _state.md, dziennik.md.
- Projekt: atomowa kolejka i przydział, fallback po proponowanych 35–55 s, stabilne nicki/persony, legalne buildy, wspólny budżet draftu 35 s, utility AI z pamięcią i opóźnioną obserwacją, kontekstowe błędy, zwykłe komendy, testy i etapowe uruchomienie lokalne.
- Weryfikacja tej sesji: przegląd dokumentów i odniesień do kodu; nie uruchamiano testów gry, migracji ani wdrożenia. Pozostało: przegląd propozycji, implementacja ośmiu etapów, testy live, strojenie i human playtests. Nie gwarantowano nierozpoznawalności AI.


## 2026-09-08 — Codex — Magic Arrow: nowy efekt Arkany

- Dodano proceduralny grot, potrójną smugę, pierścień wystrzału i krystaliczne trafienie w palecie `#64B5FF` / `#EAF4FF`. Preset castingu: Impulse, `attack_1h_01`, zgodny tint, subtelniejsza skala, bez pierścienia naziemnego.
- Integracja z cast_released, spell_impact/dodged, spell_reflected i pending_impacts; jedna instancja na action_id, śledzenie celu i skali aktora, cleanup po snapshotach, końcu meczu i disposal. Serwer nadal rozstrzyga Magic Arrow natychmiast (`travel_time: 0`); 0.14 s lotu i 0.10 s odbicia dotyczą tylko animacji.
- Pliki: `Game/FX/MagicArrow.cs/.gdshader/.tscn`, `Game/FX/_Previews/MagicArrowPreview.cs/.tscn`, `Core/Spells/ISpellEffect.cs`, `Application/Modules/Spell/Effects/SpellEffectFactory.cs`, `SpellEffectConfigurations.cs`, `SpellEffectManager.cs`, `SpellEffectManager.DuelV2.cs`, `SpellEffectManager.MagicArrow.cs`, `Resources/SpellVisuals/magic_arrow.tres`, `Game/ScenesV3/VfxTest/VfxTestScreen.cs`, nowe `VerifyMagicArrow.cs/.tscn`. W `VerifyMirrorWard.cs` próbka niezaimplementowanego zaklęcia zmieniona z Magic Arrow na Firebolt, zachowując sens kontroli po dodaniu strzały. Towarzyszące UID wygenerowane przez Godot.
- Walidacja: build 0 błędów / 9 wcześniejszych ostrzeżeń; Magic Arrow 13/13, MirrorWard 60/60, WardAudio 10/10. Test przed wdrożeniem: pięć oczekiwanych błędów braku pocisku. Review wykrył późny burst po anulowaniu; dodatkowy test odtworzył błąd, poprawiono wraz z odpinaniem callbacka celu przy disposal i ponownie sprawdzono. Logi: `/tmp/hexbane-arrow-{build,final,ward,audio,capture}.log`. GPU: pięć PNG w `verification/magic-arrow/`, wizualnie sprawdzone charge/flight/impact/reflection; brak błędów shaderów.
- Podgląd: `Game/FX/_Previews/MagicArrowPreview.tscn` (F6), 1 trafienie / 2 odbicie / 3 unik / Tab odwrócenie / Space powtórka. VfxTest także zawiera Magic Arrow.
- Notatki: spell-effect-system, spell-vfx-configuration, vfx-and-race-animation, client-architecture, duel-v2-client, _state (decyzja i rationale), dziennik. Nie zmieniano protokołu, mechanik serwera ani SFX. Bez commita.
- Poza wykonaną walidacją: rzeczywisty mecz Nakama i wydajność na fizycznym Androidzie nie były testowane.

## 2026-09-08 — Codex — rewizja balansu ras/statystyk/czarów

Porównano aktualny kod z progression, combat-stat-rules i spell-system. Raport [[2026-09-08-balance-review]]: wszystkie pary ras, trzy scenariusze progresji, kontrolne próby Arrow kontra AI/Firebolt/Heavy Bolt, razem10 860 walk. Rozpisano statcapy i legalne ekstrema, bonusy ras, skalowanie ataków/heali/shielda, ticki DEX, skille i XP. Wyniki/harness/hash źródeł w docs/audits/2026-09-08-balance-review-data; indeks uzupełniony.

Najważniejsze: Orc dominuje w testowanej rodzinie buildów; Gnome/Shadow słabe; bonusy szkół tylko Human działają; pełny flat damage wzmacnia tanie czary; maksymalne skille/progresja prowadzą do licznych timeoutów. Nie zmieniono produkcyjnego kodu ani parametrów. Propozycje są do kolejnej iteracji strojenia, nie są zatwierdzonym balansem. Pozostało: wybrać kierunek, przetestować kandydatów współczynników i buildy zoptymalizowane, potem tuning i playtest.


## 2026-09-08 — Codex — ArenaMapsDev i wspólne odtwarzanie czarów

- Użytkownik wykrył brak pocisku w ArenaMapsDev i wymagał tej samej implementacji czarów we wszystkich widokach. Przyczyna: ReferenceHud.AnimateCast odtwarzał tylko gest i osobną gałąź Mirror Reflection; nowy MagicArrowPreview i walka miały własną integrację.
- Dodano `Game/FX/SpellPresentation.cs`: wspólne kotwiczenie, skala, cel, własność węzłów i odtwarzanie czarów dla walki, ReferenceHud/ArenaMapsDev, MagicArrowPreview i ShowReflection. `SpellEffectConfigurations.Resolve` normalizuje aliasy, uwzględnia rejestr i zwraca niezależną kopię konfiguracji. Factory obsługuje rodzica/ustawienia instancji, korzystają z niego również VfxTest i FxPreview. Ścieżki scen i domyślne czasy VFX występują wyłącznie w katalogu; FxPreview ma SpellId zamiast osobnej sceny/czasu.
- Czasy/gesty castingu w podglądach pobierane przez wspólny resolver; DuelProtocol.PresentationSpell korzysta z istniejącego StandardSpells.Fallback, gdy brakuje metadanych standardów. Oba przyciski Cast w ArenaMapsDev rzucają Magic Arrow, pole id domyślnie magic_arrow; przyciski zapisanych czarów używają tego samego playbacku. Edytor profili gestów nadal służy samemu gestowi/energii dłoni.
- Zmienione pliki: SpellPresentation, SpellEffectConfigurations, SpellEffectFactory, SpellEffectManager i partials, DuelProtocol, RaceSpriteAnimator, ReferenceHud, ArenaMapsDev, VfxTestScreen, FxPreview/MirrorWardPreview.tscn, MagicArrowPreview i VerifyMagicArrow. Review wychwycił własność bariery: tworzona od razu pod chronionym aktorem, żeby przyciski trafienia ją znajdowały i nie przerywać audio przez reparenting.
- Test przed poprawką: cztery przyciski ArenaMapsDev nie tworzyły strzały. Po poprawce 25/25 kontroli (lifecycle, wszystkie cztery przyciski, obie bariery/trafienia, niezależność kopii konfiguracji i propagacja zmiany czasu lotu), build 0 błędów / 9 wcześniejszych ostrzeżeń. WardAudio 10/10, MirrorWard 60/60 podczas refaktoru. VfxTest startup PASS. GPU `verification/magic-arrow/arena-maps.png` potwierdza widoczny pocisk w właściwej scenie (panel można schować). Wymuszone zakończenie samodzielnego loop-preview przez --quit-after zgłaszało dwa zasoby w użyciu przy zamykaniu; brak błędów odtwarzania.
- Dokumentacja: spell-effect-system, spell-vfx-configuration, vfx-and-race-animation, duel-v2-client, _state i dziennik. Bez zmian protokołu ani mechanik; brak testu live Nakama/Android i bez commita.

Końcowa walidacja po ustaleniu rodzica bariery przed Play: MirrorWard ponownie 60/60; łącznie 95 kontroli PASS. Otworzono ArenaMapsDev do ręcznego podglądu.

## 2026-09-08 — Codex — plan redesignu ras/statcapów/primary

Uwzględniono doprecyzowanie użytkownika: obecny katalog to pierwsza paczka, stare rasy do przeprojektowania, Arrow1–2 dmg jako mirror breaker, dwa stałe primary i3→6/7 dodatkowych slotów. Sprawdzono walidację level-upu i obecną drabinkę slotów. Zapisano propozycję [[2026-09-08-race-primary-progression-redesign]]: wspólny budżet, softcapy zamiast zakazów rasowych, tożsamości ras, poziome warianty primary i etapy wdrożenia. Jawnie wskazano obecną różnicę Human4 na starcie i proponowane3 dla wszystkich. Zaktualizowano indeks, doprecyzowanie audytu i decyzje użytkownika w _state. Bez zmian kodu. Pozostało: wybór/akceptacja projektu, potem osobne plany wykonawcze i strojenie.


## 2026-09-08 — Codex — skill tworzenia animacji czarów

Utworzono osobisty `~/.codex/skills/hexbane-spell-animation/SKILL.md` i `agents/openai.yaml`. Zakres: natura/paleta i kontrakt serwera, projekt etapów efektu, wspólna konfiguracja/factory/SpellPresentation, zgodność z eventami i cleanup, rzeczywiste przyciski ArenaMapsDev, pozostałe preview, kontrola GPU oraz zapis do vaulta. Skill odsyła do dokumentacji; nie kopiuje katalogu ani nie narzuca wyglądu/czasu Magic Arrow innym czarom. Bez generatorów i zbędnych plików pomocniczych.

Walidacja struktury `quick_validate.py`: PASS. Niezależne zastosowanie read-only do scenariusza Poison poprawnie wybrało venom/status zamiast niepotrzebnego pocisku; wskazało brak offline metadanych niestandardów i wymaganie presetu dla przycisków zapisanych czarów. Uzupełniono skill o te warunki i różnicę między zwykłymi Cast (Magic Arrow) a testem wskazanego id. Bazowe błędy integracji, które skill adresuje, zostały odtworzone w tej sesji przed jego utworzeniem (cztery niedziałające ścieżki ArenaMapsDev, późny burst po Stop, własność bariery).

Notatki: legacy-and-tooling (lokalizacja i przykład użycia), _state, dziennik. Nie zmieniano kodu gry, nie commitowano i nie publikowano skilla. Wywołanie: `$hexbane-spell-animation zrób animację Poison`.

## 2026-09-08 — Codex — korekta planu rozwoju primary

Na podstawie uwag użytkownika przepisano sekcje primary i slotów w [[2026-09-08-race-primary-progression-redesign]]: Mirror chroni w100% przed jednym pakietem, siła odbicia25%→100% lub rozwój czasu; Arrow dwie proponowane gałęzie: tempo/koszt, bez wzrostu dmg. Zaproponowano ograniczony budżet na czar i build hybrydowy. Przywrócono w planie Human4 na starcie (zgodnie z obecnym kodem). Zapisano decyzje użytkownika w_state, rozróżniając propozycje rang i polityki statusów. Kod bez zmian. Pozostało zatwierdzenie szczegółów i implementacja z testami odbić/DoT/hex oraz progresji.

## 2026-09-08 — Codex — graf rozwoju primary,6 poziomów

Zastąpiono propozycję2 punktów grafem6 poziomów z rozwidleniami i ponownym łączeniem ścieżek. Checkpoint3 lustra odblokowuje zwracanie statusów/DoT; do tego czasu propozycja zachowuje pełną ochronę przed przechwyconym pakietem. Checkpoint6 dodaje wybór efektu końcowego. Rozpisano kandydatów słabszego startu i węzłów dla Arrow/Mirror, zasady zapisu grafu, refundów i niestakujących bonusów tempa. Zaktualizowano plan i_state; bez zmian kodu. Pozostało zatwierdzenie szczegółów/liczb i symulacja wszystkich ścieżek, szczególnie wczesnego okna Arrow kontra Mirror.


## 2026-09-08 — Codex — Fireball / Firebolt VFX

- Na podstawie skilla hexbane-spell-animation wdrożono kanoniczny `firebolt`, alias prezentacyjny `fireball`: natura Żar, `#FF713D` / `#A52E25`, gorący rdzeń, turbulentne płomienie i iskry, rozbłysk trafienia. Iteracja GPU zmniejszyła prześwietlenie i różowy odcień na fioletowej arenie; shader używa premultiplied alpha z emisją.
- Wspólny `SpellProjectile` obsługuje cykl życia Firebolt i MagicArrow; manager `SpellEffectManager.Projectiles.cs` zastępuje partial MagicArrow i rozpoznaje pole IsProjectile katalogu. Domyślny lot 0.18 s, odbicie 0.12 s, ogon trafienia 0.62 s — czysto kosmetyczne, bez zmiany serwerowego travel_time=0 i natychmiastowych obrażeń.
- `SpellPresentationCatalog` normalizuje fireball→firebolt i zawiera zgodny z YAML fallback Firebolt (cast1/recovery0.4/mana9); serwer ma pierwszeństwo. Jeden preset `firebolt.tres` zastępuje fireball.tres, zachowuje attack_2h_02 i nadaje profil Embers/paletę. Save/load ArenaMaps normalizuje alias, obie nazwy działają przez zapisany czar. VfxTest dodaje Firebolt, prawidłową etykietę Projectile, wspólne Play/Clear i zapis per-spell właściwości override. Wspólny `ProjectilePreview` obsługuje sceny MagicArrowPreview oraz nowy FireboltPreview.
- Pliki: Game/FX/{Firebolt.cs,Firebolt.gdshader,Firebolt.tscn,SpellProjectile.cs,MagicArrow.cs,MagicArrow.gdshader,SpellPresentation.cs}; Game/FX/_Previews/{ProjectilePreview.cs,MagicArrowPreview.tscn,FireboltPreview.tscn}; Core/Spells/{SpellPresentationCatalog.cs,SpellEffectRegistry.cs}; Application/Match/DuelProtocol.cs; Application/Modules/Spell/Effects/{SpellEffectConfigurations.cs,SpellEffectManager.cs,SpellEffectManager.DuelV2.cs,SpellEffectManager.Projectiles.cs}; SpellVisualPreset.cs, ArenaMapsDev.cs, VfxTestScreen.cs, firebolt.tres, VerifyFirebolt.cs/.tscn. UID przeniesione przy zmianach nazw. VerifyMirrorWard używa heavy_bolt jako nadal niezaimplementowanej próbki (Firebolt przestał nią być).
- Walidacja: pierwotny test katalogu FAIL; po implementacji Firebolt 25/25, MagicArrow 25/25, MirrorWard 60/60. Sprawdzone eventy/snapshoty, duplicate, reflection/dodge, cancellation/disposal, alias/metadane, przyciski zapisanych czarów obu stron, wspólne parametry i rzeczywisty kafelek/Clear w VfxTest. Build 0 błędów, 9 istniejących ostrzeżeń. Review niezależny read-only: brak uwag. GPU shader kompiluje się; sześć obrazów w verification/firebolt/, sprawdzono lot, trafienie i ArenaMaps. Logi /tmp/hexbane-fire-*.log.
- Notatki: spell-effect-system, spell-vfx-configuration, vfx-and-race-animation, duel-v2-client, client-architecture, _state, dziennik. Skill doprecyzowano, żeby sprawdzał aktualne pokrycie fallbacków zamiast zakładać wyłącznie standardy; quick_validate PASS.
- Bez nowych SFX, zmian protokołu/mechanik serwera i commita. Nie testowano meczu live Nakama ani fizycznego Androida. Podgląd: FireboltPreview.tscn; w ArenaMapsDev wpisać fireball/firebolt, Wczytaj preset, Rzuć zapisany czar.

## 2026-09-08 — Codex — XP i wejście do rankedów

Przygotowano [[2026-09-08-xp-ranked-progression]]: kandydat kosztu kolejnego poziomu50+10*(L−2),5510 XP do30,120/90 XP za wynik, ok.53 meczów przy50% zwycięstw. Zweryfikowano sumy krzywej. Uwzględniono doprecyzowanie użytkownika: skille/MP nie blokują rankedów i rozwijają się dalej, bez automatycznej normalizacji/maxowania skilli. Wskazano konieczność nowego źródła MP po30, testów różnic siły i migracji starego XP. Dodano link w planie grafów, indeks i decyzję użytkownika. Kod i baza bez zmian; współczynniki i zasady ekonomii pozostają propozycją.

## 2026-09-08 — Codex — ekonomia MP pod draft

Uwzględniono cel większej kolekcji niż slotów (6/10/15 wobec4/5/6). Przeliczono bazowy budżet przy cenie5 MP:15/35/60 do poziomów4/8/12, potem2 MP/level do96 na30. Zaproponowano po capie500 XP postępu→5 MP, bez dodatkowych leveli, oraz jednorazowe+5 MP za progi25/50/75/100 każdego skilla. Zaznaczono limit bieżącej paczki12 opcjonalnych czarów, brak gate'u ranked i brak podwójnego naliczania nagród. Zaktualizowano oba plany i_state. Kod bez zmian. Pozostało strojenie podaży względem cen/paczek oraz testy progów i migracji.

## 2026-09-08 — Codex — korekta czasu levelowania do80 meczów

Zaktualizowano [[2026-09-08-xp-ranked-progression]]: proponowany koszt kolejnego poziomu50+17*(L−2),8352 XP do30. Przy120/90 XP i40 wygranych/40 przegranych suma8400 osiąga cap. Zweryfikowano próg każdego poziomu, końcowy koszt526, skrajne70 zwycięstw/93 porażki, czasy slotów i checkpointu primary3. Decyzja użytkownika zapisana w_state. Nagrody MP, dalsze skille i brak ich gate'u ranked pozostają bez zmian. Kod bez zmian.

## 2026-09-08 — Codex — wolniejsze sloty i65 gier do capu

Przeliczono propozycję XP:50+13*(L−2),6728 XP do30;64 gry przy równym bilansie dają6720, więc kolejna osiąga cap. Sloty proponowane na poziomach7/11/16 odpowiadają około4,7/10,3/20,1 gier. MP przesunięto do15/35/60 przy tych progach,88 do30; stały dochód po capie i skill milestone bonuses bez zmian. Zaktualizowano plan XP, drabinkę w planie primary i_state. Kod bez zmian.

## 2026-09-08 — Codex — obniżenie XP za porażkę

W planie XP zmieniono nagrody na120/70 i przeliczono krzywą45+12*(L−2),6177 XP do30. Przy50% zwycięstw średnia95 XP daje około65 gier; skrajnie52 zwycięstwa lub89 porażek. Sloty7/11/16 pozostają około5/10/20 gier. Uwzględniono wolniejsze zasilanie post-cap MP (500 XP to około5,3 meczu). Kod bez zmian.


## 2026-09-08 — Codex — implementacja ras, grafów primary i progresji

Wykonano zatwierdzone zmiany backendu oraz klienta: wspólny budżet/statsoftcap/respec,6ras, grafy primary i ich runtime, krzywa6177XP/120:70, sloty7/11/16, MPstudy+skillmilestones, rankedlevel30. Transakcyjne nagrody z receipt/retry i blokada buildu podczas meczu. Migracje000004/000005 z ochroną istniejących danych i jawnym guardem rollbacku nowych postaci. Pliki: modules/{primary,progression,character,combat,race,skills,match,spell_system}, YAMLprimary, cmd/duel-sim, db/migrations;13plików klienta opisane w [[2026-09-08-redesign-implementation]]. Dokumentacja: server/progression,combat-stat-rules,spell-system; protocol/character-details,shared-types,rpcs,combat-v2; client/duel-v2-client;2plany i ledger, _index, _state, dziennik.

Walidacja: go test ./..., go test -race ./..., go vet ./..., Linux makebuild; klient build0errors/9existingwarnings i testy Progression PASS. DisposablePG17.6 up/down/up i integracje migracji/8równoległych rozliczeń PASS.1000symulacji bez błędów zasobów,45.4%timeoutów. Pozostało strojenie tempa/playtest, livewalidacja nowych UI i wdrożenie zgodnych migracji+binarek. Brak commitów i zmian na działającej bazie; wcześniejsze niezacommitowane zmiany zachowane.

## 2026-09-08 — Claude Code — UI progresji po stronie klienta (redesign duel_v2.4)

Zaimplementowano docelowe UI dla wdrożonego serwerowo redesignu (zamiast surowych popupów): ekran postaci ma 4. zakładkę **Primary** (drabinka poziomów 1/5/7/10/11/16/23/30 z tierami primary, slotami i ranked; dwa edytory grafu DAG z chipami węzłów, stanami locked/checkpoint/selected, podglądem „Pending/Saved” z foldu modyfikatorów, SAVE PATH/REVERT), tryb **Reallocate all points** inline na Stats (pula = 400+5/lvl + nierozdane, min 10, hold-to-repeat na +/−, złoty softcap >200, podgląd różnic), study XP w nagłówku na capie, znacznik RANKED w meta, cechy rasy pod nazwą, podpis spellbooka (MP/kolekcja/sloty/następne MP), przycisk „Primary · tier N › Develop” dla standardów. Dashboard: „Level N · XP to next”/„Level 30 · Ranked”, panel **Primary Path Ready** (skrót do zakładki), mniejsza czcionka 3‑cyfrowych odznak. ModeOverlay: trzy przyciski w scenie (vs Player / Ranked 🔒 / vs AI) + linia z powodem blokady. GameOver: +MP, slot tylko przy przekroczeniu 7/11/16, „primary tier N”, „RANKED UNLOCKED”. Core: `ProgressionCurve` (krzywa 6177, MP, sloty, tiery, budżet) i `PrimaryPath` (teksty węzłów, baseline, fold) z testami w `Tests/Progression` (PASS). DTO: `study_xp/ranked_eligible/primary_tier`, `spells_learned/spell_slots_unlocked`, pełny `config` i `modifier` grafu.

Weryfikacja: `dotnet build` 0 błędów; harness `Game/ScenesV3/Dev/ProgressionVerification.tscn` (login kontem testowym, zrzuty PNG, zapis ścieżki i respec) na 1360×612 i 1920×1080 dla human L1 / elf L12 / orc L30 — ścieżka lustra i respec trafiły do DB. Środowisko: lokalna Nakama miała nowy plugin, ale bazę na migracji 3 i pusty stan — zastosowano `make migrate-up` (000004/000005), `make db-seed`, a po restarcie kontenera skopiowano pliki migracji do `/nakama/migrations` (obraz ich nie zawiera — do przebudowy obrazu). Konta elf/orc podbite SQL‑em do L12/L30 na potrzeby podglądu (sloty w DB nie przeliczone). Bez commita; w repo jest równolegle cudzy nieśledzony `Game/ScenesV3/Dev/ArenaMaps/`.

Notatki: duel-v2-client (sekcja „Catalog2.4 progression UI”), character-details (DTO), _state (decyzja), dziennik. Pozostało: test na fizycznym Androidzie, przebudowa obrazu Nakamy z migracjami, ewentualne skrócenie długiej linii statusu drabinki na bardzo wąskich ekranach.



## 2026-09-08 — Komplet animacji zaklęć i zasłanianie przez postać

- Dodano jedenaście brakujących animacji: Heavy Bolt, Delayed Hex, Poison, Paralysis, Cleanse, Mend, Greater Heal, Regeneration, Barrier, Dispel, Consume Venom. Katalog ma 14/14 pełnych efektów. Formy i palety wynikają z natury/mechaniki; projectile pozostaje dozwolony, nie obowiązkowy.
- Pliki klienta: `Game/FX/SpellField.cs/.gdshader` i 11 scen; 11 `Resources/SpellVisuals` presetów; konfiguracje, `SpellPresentation`, katalog offline; `SpellEffectManager.Fields.cs` i lifecycle; VfxTest zapisuje parametry do pojedynczej sceny. Aktualizacja verifiera MirrorWard używa rzeczywiście nieznanych id zamiast nowo zaimplementowanych statusów.
- Zdarzenia i snapshoty współdzielą statusy po instance id. Obsłużono opóźniony damage klątwy, usunięcie bez fikcyjnego wybuchu, pochłanianie bariery, no-op Consume Venom, dodge oraz czyszczenie. Tył/przód efektu renderuje się odpowiednio pod/nad aktualnym sprite’em postaci.
- Testy: build 0 błędów/9 wcześniejszych ostrzeżeń; VerifySpellFields GPU 146/146 (oba przyciski ArenaMapsDev dla 11 czarów, VfxTest tile/swap/clear, lifecycle i 24 pikselowe asercje okluzji 6 ras × 2 strony); Magic Arrow 25/25, Firebolt 25/25, MirrorWard GPU 60/60. Obejrzano zrzuty `verification/spell-fields/`. Przegląd kodu zakończony po poprawkach dwóch usterek.
- Notatki: `spell-effect-system`, `spell-vfx-configuration`, `vfx-and-race-animation`, plan `2026-09-08-remaining-spell-vfx`, `_index`, `_state`.
- Pozostało: rzeczywisty mecz Nakama i pomiar na urządzeniu Android. Nie dodawano nowych SFX. Nie zmieniano mechanik ani balansu serwera.
