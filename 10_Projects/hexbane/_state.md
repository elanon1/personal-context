---
type: project
project: Hexbane
domain: [projects, creative]
status: active
state: active
repo: https://github.com/elanon1/hexbane
created: 2026-08-31
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, gamedev, godot, csharp, nakama, go, kubernetes, ai-art]
aliases: [hexbane, hexbane-server]
---

# Hexbane — State

> **Dokumentacja techniczna projektu (klient, serwer, protokół, infra, plany) jest tylko tutaj:
> [[_index]] → `10_Projects/hexbane/docs/`.** Repozytoria nie mają własnych docs (sztywna reguła w ich
> `CLAUDE.md` / `AGENTS.md`). Dziennik pracy: [[dziennik]].

## Summary

**Hexbane** to gra 1v1 (desktop + Android): pojedynek magów w czasie rzeczywistym, bez ruchu — liczy
się wybór zaklęć, kolejkowanie akcji (cast time + recovery, brak cooldownów) i medytacja
(regeneracja many). Dwa repozytoria, oba w trakcie **niezacommitowanego** przeprojektowania
„duel_v2” (wrzesień 2026):

- **Klient** — `~/RiderProjects/hexbane` (`elanon1/hexbane`, branch `feat/duel-v2-client` = `master`
  + niezacommitowana praca): Godot 4.5.2 + C# .NET 9, Nakama SDK 3.16, warstwy
  `Game → Application → Core`, DI + własny CQRS. → [[client-architecture]], [[duel-v2-client]]
- **Serwer** — `~/GolandProjects/hexbane-server` (`elanon1/hexbane-server`, branch
  `feat/spell-system-redesign` = `main` + niezacommitowana praca): plugin Go do Nakama 3.27
  (`backend.so`), Postgres 17. → [[server-architecture]], [[dev-setup]]
- **Reguły gry (stan duel_v2):** 6 ras, 14 zaklęć (2 standardowe zawsze dostępne + 6 starterów, z
  których przy tworzeniu postaci wybiera się 3, Human 4), HP/mana ze statystyk i ras, aktywne
  skille, regeneracje i odporności (szczegóły i ograniczenia metadanych: [[combat-stat-rules]]),
  katalog `duel_v2.3`, sloty draftu 3→6 (Human 4→7), tick 100 ms, mecz 180 s.
  → [[progression]], [[spell-system]], [[combat-v2]]
- **Protokół:** opcode’y 0–10, 16, 70, 50, 199 bez zmian; walka to 29 (komenda) → 30 (prywatny
  wynik) / 31 (zdarzenia) / 32 (snapshot). Opcode’y 11–15 i 21–28 **wycofane**. → [[opcodes]], [[rpcs]]
- **Infra:** obecnie **tylko lokalnie** (docker compose, `NAKAMA_SERVER=local`); chart Helm + Argo CD
  na własny k8s (`hexbane.elanon.pl`) istnieją, ale nie są celem prac. → [[infra-and-deploy]]

Decyzja fundamentalna (26.06.2025): klient przepisany z GDScript na C#.

## Status

`active`. Pętla rdzeniowa działa end-to-end na lokalnym stacku: logowanie (Google przez przeglądarkę
lub e-mail) → lokalny tutorial → kreator postaci (5 kroków, wybór 3/4 starterów) → matchmaking
(PvP `normal` / bot `ai_duel`) → lobby z draftem (35 s/tura) → pojedynek duel_v2 na scenie
`ReferenceDuel/MainReference.tscn` z prawdziwymi sprite’ami ras → game over z XP/level-upem.

**Największe ryzyko:** cała praca z 2026-09 w obu repo jest niezacommitowana (serwer: ~276 wpisów w
`git status`, w tym skasowana stara historia migracji `000001..000016` zastąpiona świeżym baseline
`000001..000003`). Snapshot repo: [[repos-and-branches]].

**Content:** 13 zachowanych ikon pokrywa 14 obecnych zaklęć; VFX obejmuje wszystkie 14 zaklęć: Mirror Reflection z przywróconymi dźwiękami, Magic Arrow i Firebolt/Fireball ze wspólnym cyklem życia pocisku oraz 11 różnorodnych efektów na postaci z warstwami przed/za sylwetką. Nowe animacje castingu i presety zachowane. Stare VFX/SFX i assety generatora usunięte 2026-09-08. → [[spell-vfx-configuration]]

## Decisions log

### 2026-09-08 — Mobilne menu po redesignie

Zachować styl gry, dopasowywać układ do szerokości i wysokości; długie sekcje przewijać, a nawigację kreatora/postaci/dashboardu pozostawić na ekranie. Usunąć prezentację wycofanych limitów ras. **Why:** skalowanie wyłącznie według wysokości i przywracanie desktopowych marginesów po wyborze powodowały przepełnienie; dalsze zmniejszanie tekstu pogarszałoby czytelność. Weryfikacja i ograniczenia: [[2026-09-08-mobile-layout-review]].


- **2026-09-08 — Tutorial produces the shared VFX event contract.** Canonical releases, stable action ids, complete impacts and status lifecycle replace incompatible local emissions. **Why:** using the same scene/factory is insufficient if the tutorial never supplies the events those effects require. Show the last impact before the next modal, while keeping simulation/input paused. Cleanse uses a gesture with adequate pre-release frames for every race.

- **2026-09-08 — Complete spell VFX catalog with actor depth layers.** Eleven remaining spells use distinct field forms and canonical scenes/presets; Arrow/Firebolt remain projectiles. **Why:** geometry should express mechanics/nature, and every view must resolve the same implementation. Rear/front passes surround the animated sprite to prevent effects behind the body shining through it. Authoritative persistent effects await server removal so late hex damage can still detonate.


- **2026-09-08 — Klient progresji: 4. zakładka `Primary` + inline respec, bez popupów.** Ekran postaci dostał zakładkę Primary (drabinka poziomów + dwa edytory grafu z podglądem „Pending/Saved” liczonym po stronie klienta z modyfikatorów węzłów) oraz tryb „Reallocate all points” na tych samych wierszach Stats; ranked jako trzeci przycisk w scenie ModeOverlay z blokadą i podpowiedzią; dashboard pokazuje „Primary Path Ready”. **Why:** użytkownik chce widoczności i możliwości progresji w tym samym moodzie i skalowalnie pod telefon poziomy — okienka `Window` z surowymi SpinBoxami z pierwszej implementacji nie przechodziły przez ResponsiveLayout ani theme. Zweryfikowano live (zapis ścieżki i respec trafiły do DB) na 1360×612 i 1920×1080. Szczegóły: [[duel-v2-client]].

- **2026-09-08 — Implementacja przebudowy zatwierdzona i wykonana w kodzie obu repo.** Wspólne statystyki/softcap, nowe cechy6ras, primary DAG6tierów, XP120/70 i6177do30, sloty7/11/16, MPpo capie+milestones, ranked od30. **Why:** użytkownik polecił wdrożyć wszystkie ustalenia; rasy muszą działać z bieżącą paczką i pozwalać na swobodne buildy. Szczegółowe rozstrzygnięcia i wyniki: [[2026-09-08-redesign-implementation]]. Testy/race/vet/buildy i migracje testowe PASS; bez deploya.45.4%timeoutów w symulacji wymaga dalszego strojenia, nie oznacza gotowego balansu.


- **2026-09-08 — Porażka70 XP.** Użytkownik obniżył proponowane XP za porażkę z90 do70. **Why:** zwycięstwo powinno być wyraźniej premiowane. Wygrana pozostaje120, remis proponowany jak porażka; nowa proponowana krzywa6177 XP utrzymuje około65 gier przy50% zwycięstw oraz późniejsze sloty.

- **2026-09-08 — Wolniejsze sloty, około65 meczów do capu.** Użytkownik ustalił slot4 po około4–5 grach, slot5 po10, slot6 po20 oraz skrócił drogę do maksymalnego poziomu do około65 meczów. **Why:** dłużej rozwijać draft, ale szybciej dojść do końca levelowania. Proponowane progi7/11/16 i6728 XP do30 są w [[2026-09-08-xp-ranked-progression]]; MP dopasowane do nowych progów. Human zachowuje dodatkowy slot od startu.

- **2026-09-08 — Około80 meczów do maksymalnego poziomu.** Użytkownik ustalił docelowy czas progresji na około80 rozegranych meczów do max levelu. **Why:** szybki początek ma przejść w dłuższy rozwój przed rankedami. Propozycja krzywej8352 XP przy120/90 XP daje około80 gier przy50% zwycięstw; zastępuje wcześniejszy cel53 gier. Rozwój MP/skilli po capie pozostaje niezależny.

- **2026-09-08 — Draft wymaga nadmiaru poznanych czarów.** Użytkownik odrzucił model1 darmowego czaru na nowy slot; cel to orientacyjnie6/10/15 poznanych zaklęć przy4/5/6 slotach, zależnie od zakupów za MP. **Why:** draft ma wymuszać rezygnację z części dostępnych opcji. Zwiększone wczesne MP, dochód po capie i bonusy za progi skilli są propozycjami w [[2026-09-08-xp-ranked-progression]], nie zatwierdzonymi wypłatami.

- **2026-09-08 — Skille i MP nie blokują rankedów.** Użytkownik chce szybkie początkowe poziomy, stopniowe spowolnienie i osiągalny szybko maksymalny poziom postaci. Rozwój skilli do capu i zdobywanie MP mają trwać niezależnie od wejścia do rankedów. **Why:** ranked ma być dostępnym etapem gry, a nie nagrodą za ukończenie całego grindu. Krzywa5510 XP, nagrody120/90 i próg30 to propozycje w [[2026-09-08-xp-ranked-progression]], nie wdrożone reguły.

- **2026-09-08 — Fireball jako alias Firebolt, wspólny cykl życia pocisków.** Nowy ognisty efekt natury Żar; `SpellProjectile` i wspólny manager obsługują oba pociski, `ProjectilePreview` oba podglądy. Jeden preset `firebolt.tres` oraz metadane offline zgodne z YAML. **Why:** zachowanie nomenklatury użytkownika bez rozdzielania efektów i konfiguracji między widokami; `travel_time: 0` pozostaje decyzją serwera.

- **2026-09-08 — Primary jako graf6 poziomów.** Użytkownik chce słabszy start, rozwidlenia oraz dodatkowe efekty na checkpointach3 i6; odbijanie osłabień i czarów okresowych lustrem dopiero od3 poziomu. **Why:** rozwój ma otwierać nowe możliwości i różnicować buildy. Nowe wartości startowe, bonusy węzłów i efekty końcowe pozostają propozycjami w [[2026-09-08-race-primary-progression-redesign]].

- **2026-09-08 — Rozwój primary i start Humana.** Użytkownik wybrał rozwój Mirror: pełna ochrona przed przechwyconym trafieniem od startu, odbijane obrażenia25%→100% oraz druga ścieżka czasu trwania. Arrow również ma dostać dwie ścieżki. Human zaczyna z4 slotami. **Why:** rozwój primary ma zwiększać różnorodność buildów, a dodatkowy slot jest cechą Humana od początku. Budżety punktów i konkretne rangi to nadal propozycja w [[2026-09-08-race-primary-progression-redesign]].

- **2026-09-08 — Skill `hexbane-spell-animation`.** Proces tworzenia VFX zapisany jako osobisty skill Codex w `~/.codex/skills/`, z odsyłaczami do kontraktów vaulta zamiast kopii katalogu czarów. **Why:** powtarzalne tworzenie efektów zgodnych z naturą i wspólnym playbackiem, z kontrolą rzeczywistych widoków użytkownika.

- **2026-09-08 — Kontrakt kolejnych paczek i primary spelli.** Użytkownik doprecyzował, że14 czarów to pierwsza paczka; rasy wymagają redesignu po starym katalogu. Arrow służy głównie do rozbijania luster (docelowo1–2 dmg), Mirror i Arrow pozostają zawsze dostępnymi primary. **Why:** balans i rasy muszą wspierać przyszłe paczki oraz stały podstawowy zestaw kontr. Zmiany statcapów i warianty primary są dopiero propozycją w [[2026-09-08-race-primary-progression-redesign]], nie zatwierdzoną implementacją.

- **2026-09-08 — Jeden katalog i wspólne odtwarzanie VFX we wszystkich widokach.** Walka i podglądy korzystają z `SpellEffectConfigurations.Resolve` / `SpellEffectFactory`, a widoki z aktorami z `SpellPresentation`; czasy castingu podglądu pochodzą ze wspólnego resolvera metadanych. **Why:** użytkownik wykrył brak Magic Arrow w ArenaMapsDev i wymaga eliminacji rozjazdów między widokami.

- **2026-09-08 — Propozycja fallback AI: kolejka serwerowa i wspólny silnik walki.** Przygotowano [[2026-09-08-fallback-player-design]] i [[2026-09-08-fallback-player-plan]], bez wdrożenia. Rekomendacja: trwałe fikcyjne persony, legalne buildy, opóźniony draft, AI utility z ograniczoną obserwacją i kontekstowymi błędami; roboczo zwykła kolejka po 35–55 s. **Why:** obecny bot ma stałą tożsamość i natychmiastowe wybory, a timeout po stronie klienta ryzykuje podwójny przydział meczu. To kierunek projektowy do przeglądu, nie zatwierdzona zmiana zachowania.

- **2026-09-08 — Magic Arrow: błękitna Arkana i krótki przelot wizualny.** Nowy proceduralny shader używa `#64B5FF` / `#EAF4FF`; efekt lotu trwa 0.14 s, odbicia 0.10 s, bez zmian mechanik. **Why:** natura `arcana` wymaga kryształu i geometrii, a serwer ma `travel_time: 0`; prezentacja nie może opóźniać obrażeń ani zmieniać kolejności zdarzeń.

- **2026-09-08 — Przywrócono statystyki, rasy i skille do walki.** Wspólny profil zasila mecz, efekty, kartę postaci i symulator; klient otrzymuje efektywne koszty/czasy również w drafcie. Zachowano MatchLog i wcześniejsze czyszczenie helperów/bufora/powiadomień. **Why:** użytkownik wyraźnie zlecił przywrócenie mechanik („przywroc”); ich nieużywanie było regresją integracji. Nie dopisujemy szkół obrażeń z lore — katalog nadal neutralny, ograniczenia opisuje [[combat-stat-rules]].

- **2026-09-08 — Przywrócenie SFX Mirror Reflection.** Przywrócono oryginalne formation.wav/shatter.wav i WardAudio; reszta usuniętych SFX pozostaje wycofana. **Why:** użytkownik doprecyzował wyjątek od czyszczenia dźwięków.


- **2026-09-08 — Czyszczenie klienta: zachowujemy nowy casting i Mirror Reflection.** Usunięto stare VFX, wszystkie SFX (również bariery), klientowy storytelling, nieużywane handlery starej walki i GTweens. Zachowano aktualny katalog mechanik, ikony używane przez HUD, animacje ras, gesty, medytację i muzykę.
  **Why:** użytkownik chce przygotować czystą bazę pod nowe efekty zaklęć; obecne ikony i mechaniki są nadal używane, a storytelling jest osobnym wycofanym prototypem. Zakres tej sesji to repo klienta; nie zmieniano serwera.


- **2026-09-08 — Zakres czyszczenia serwera po audycie.** Usuwamy nieużywane helpery/walidatory,
  stare `CastInterruptions` oraz niepodłączone helpery powiadomień; `MatchLog` zostaje.
  `get_progression` naprawione na wspólne reguły XP i slotów `{4,8,12}`. Odporności, regeneracja
  i skille mają wrócić: ich kod i testy zachowujemy; zakres aktywacji wymaga doprecyzowania.
  **Why:** nieużywany kod zawiera zarówno zbędne pozostałości, jak i docelowe mechaniki gry;
  użytkownik rozdzielił te kategorie. Nie należy kasować całego starego modelu walki na podstawie
  braku bieżących wywołań. → [[2026-09-08-server-unused-code-audit]].

- **2026-09-07 — Jedno źródło prawdy dla dokumentacji: vault Obsidian, nie repozytoria.** Oba drzewa
  `docs/` (klient 45 plików + kopie docs serwera, serwer 75 plików + `RPCs.md`) zweryfikowane z kodem
  przez 8 agentów, scalone do `10_Projects/hexbane/docs/{protocol,server,client,infra,plans,audits}`,
  usunięte z repo (kopia w `~/hexbane-archive/docs-2026-09-07/`). Sztywna reguła READ/WRITE/LOG/NEVER
  w `CLAUDE.md` i `AGENTS.md` obu repo; `.claude/commands/opcodes.md`, `.claude/agents/ui-designer.md`
  i serwerowy `opcode-docs.md` przepięte na ścieżki vaulta. Zrzuty weryfikacyjne (804 MB PNG/GIF)
  przeniesione z `docs/client/` do `verification/` (gitignore), 7 skryptów `Verify*.cs` zaktualizowanych.
  Stare polskie notatki z 2026-08-31 → `99_Archive/Projects/hexbane-notes-2026-08-31/`. Raporty
  rozbieżności → `docs/audits/` ([[2026-09-07-server-match-audit]] i sąsiednie). **Why:** dwie
  dokumentacje rozjechały się z kodem i ze sobą (serwer zredukował docs do stubów „superseded”,
  klient trzymał stare pełne wersje; GameCountdown 9 vs 16; 63 vs 14 zaklęć; 35 vs 6 ras). Vault jest
  repo gitowym, więc docs są wersjonowane; Codex i Claude Code piszą tam zwykłymi narzędziami
  plikowymi, MCP Obsidian jest opcjonalny.
- **2026-09-06 — Tutorial lokalny w kliencie, bez meczu tutorialowego.** RPC `tutorial`
  (`status` / `complete_training` / `complete_progression`) trzyma flagi w `account_tutorials`;
  szkolenie nie daje XP/MP; stare `set_tutorial_completed` zawsze błąd. → [[server-tutorial]],
  [[client-tutorial]]. **Why:** prościej i bez kosztu meczu na serwerze.
- **2026-09-05/06 — Przeprojektowanie systemu zaklęć i walki: `duel_v2`, protokół 2, katalog
  `duel_v2.2`.** 14 zaklęć w płaskim `data/spells/*.yaml` (zamiast 63 w szkołach/tierach), stałe
  200 HP / 100 many, kolejka jednej akcji, automatyczne release + recovery, tick 100 ms, mecz 180 s,
  remis po czasie; opcode’y 29–32; snapshot co 2 ticki. Tworzenie postaci wymaga wyboru 3 starterów
  (Human 4) z 6; reszta po 5 MP. Baza zresetowana do baseline `000001..000003`. → [[spell-system]],
  [[combat-v2]], [[database]]. **Why:** stary system (63 zaklęć, formuły ze statów, 4 typy efektów bez
  handlerów) był niebalansowalny i częściowo martwy; nowy jest deterministyczny i testowalny
  (`cmd/duel-sim`, testy scenariuszowe, `scripts/test_combat_runtime.mjs`).
- **2026-09-05 — Logowanie: tylko konto Google (+ e-mail), Play Games porzucone.** Jeden przepływ
  przeglądarkowy (RFC 8252 loopback + PKCE) na PC i Androidzie, token do wbudowanego
  `AuthenticateGoogleAsync` Nakamy, bez własnego RPC. Sekret klienta typu Desktop app **świadomie**
  w `project.godot` (Google go i tak dystrybuuje; PKCE zabezpiecza wymianę). → [[google-auth]],
  [[social-sign-in]]. **Why:** Nakama nie sprawdza `aud`; Play Games wymagało osobnej konfiguracji i
  psuło build APK.
- **2026-09-04 — Zaklęcia standardowe + drabinka slotów 3–6.** `magic_arrow` i `mirror_reflection`
  zawsze dostępne poza draftem; sloty odblokowywane na poziomach 4/8/12; Human +1 slot. **Why:** każdy
  gracz ma zawsze atak i obronę, a draft nie skaluje się do 10 slotów.
- **2026-09-02 — Nowy roster: 6 ras (human, elf, dark_elf, shadow, gnome, orc)** z widełkami statów
  efektywnych (min/max, 0 = brak limitu) i traitami; domyślna rasa `human`. Sztuka: modele Tripo →
  Mixamo → Blender → sprite sheety HD/SD (`build_races.sh`), 13 klipów + 3 medytacyjne. → [[progression]],
  [[assets-pipeline]]. **Why:** mały, czytelny roster zamiast 35 ras AI-artu; traity poza slotem Human
  są dziś **martwe** (walka bez bonusów).
- **2026-08-31 — Ekran logowania przerobiony 1:1 pod referencję** (karta z poświatą, Cinzel
  Decorative, 9-patch przycisk). Gotcha: 9-patch z wypalonym napisem. Później (09-01..09-02)
  tak samo przebudowane: kreator postaci, dashboard, lobby, szczegóły postaci, news/settings/social.
- **2026-08-31 — Wycofany cały roster 35 ras; nowe rasy od zera.** Stara sztuka w
  `~/hexbane-archive/Races-2026-08-31/` (922 MB, nigdy w gicie). **Why:** „powywalaj istniejące rasy,
  będziemy robić nowe”. Migracja `000011_clear_races` nigdy nie powstała — zastąpiona resetem bazy 09-05.
- **2026-08-31 — „Na razie działamy tylko na lokalu.”** Prod/Argo/Synology poza zakresem; migracje
  muszą być bezpieczne tylko lokalnie.
- **2026-06 — Produkcja ras zautomatyzowana (skill `race-maker`).** Skill istnieje, ale shipowany
  roster powstał w pipeline Blender/Mixamo, nie z chroma-sheetów ChatGPT.
- **2026-06-18 — Animacje ras jako sprite-sheety, nie rig kostny.** Potwierdzone i rozszerzone:
  `frames.tres` ma 16 klipów, plus `hand_tracks.tres` / `meditation_tracks.tres` do kotwiczenia VFX.
- **2026-01-30 — Serwer: docs v2.** Zastąpione 2026-09-07 przez vault.
- **2025-08-12 — Serwer: „mecz prowadzony przez jednego gracza” dla tworzenia postaci (Endless
  Story).** Why: stan sesji w meczu Nakamy zamiast w RPC. Dziś martwe (patrz Open questions).
- **2025-07-29 — `PlayerState` pod mutexem, atomowy `TryCastSpell`.** Dziś: Nakama serializuje
  handlery per mecz, mutexy są „defensywne”; walka ma jedną kolejkę akcji.
- **2025-07-27 — Efekty: brak stackowania tego samego zaklęcia.** Zastąpione kolejką efektów duel_v2.
- **2025-06-26 — Klient przepisany z GDScript na C#.** Why: GDScript to nisza; C# daje perspektywę
  migracji na inne silniki.

## Open questions

- **Commit pracy z 2026-09** w obu repo (branch `feat/duel-v2-client`, `feat/spell-system-redesign`)
  — kiedy i jak (jeden PR? squash?). Push serwera na `main` = Argo auto-sync + `migrate-custom` na
  bazie z historią `000016` → nie ma ścieżki upgrade’u, tylko reset.
- **Endless Story**: moduł `endless_story` wciąż zarejestrowany (mecz `v2_create_character`, RPC
  `create_character_match_story` woła nieistniejący mecz `create_character`), `start_story`
  zakomentowane, klient nadal je woła. Usunąć razem z `OPENAI_*` w `.env.dist`?
- **Martwy kod po redesignie**: traity ras (7 z 8 kluczy), `modules/combat` (formuły), `modules/skills`
  (gain), `get_progression` (zła matematyka), `set_tutorial_completed` (zawsze błąd), `MatchLog`
  zapisywany i wyrzucany co tick, `get_users` wołane przez klienta bez RPC na serwerze; w kliencie
  stary `GameHud/**` i lokalne kompatybilne DTO castingu. Handlery wycofanych opcode’ów, martwe trasy i GTweens usunięto 2026-09-08. Klient pokazuje traity ras, choć nie działają.
- **Bezpieczeństwo protokołu**: opcode 2 w `game_countdown`/`lobby` ufa `user_id` z payloadu; opcode
  10 wysyła listę zaklęć przeciwnika jako „prywatny” widok. Błąd czy akceptowalne w 1v1?
- **Sekrety w repo** (serwer `.env.dist` klucz OpenAI, `helm/hexbane/values.yaml` PAT GitHub,
  klient `.env` pakowany do builda) — nadal obecne, nie rotowane.
- **VFX — walidacja urządzeń i serwera live**: wszystkie 14 zaklęć mają animacje i wspólny playback, sprawdzony w lokalnych fixture’ach oraz GPU na desktopie. Pozostały testy rzeczywistego pojedynku Nakama i wydajności na Androidzie. SFX poza Mirror Reflection nadal nie są zaimplementowane. → [[spell-vfx-configuration]]
- **`visual_key`** — klient czyta, serwer nie wysyła (backend do zrobienia). → [[spell-visual-key]]
- **Baseline balansu** `balance-v2.json` opisuje `duel_v2.1`, kod ma `duel_v2.2` — przeliczyć?
- Czy prod na k8s ma żyć (chart + Argo istnieją, nic ich nie opisuje poza [[infra-and-deploy]])?
- Elementy/szkoły: wszystkie 14 zaklęć ma `school: neutral`; kolumny żywiołów ras są historyczne.
  Wracają czy do usunięcia z kontraktu?

## Links

- [[_index]] — mapa dokumentacji · [[dziennik]] — dziennik pracy · [[NOW]] · [[tools-stack]]
- [[10_Projects/cluster-agent/_state|Cluster Agent]] — ten sam klaster k8s
- Repo klient: `github.com/elanon1/hexbane` · serwer: `github.com/elanon1/hexbane-server`
- Obraz: `ghcr.io/elanon1/hexbane-server:latest` · prod (nieużywany): `https://hexbane.elanon.pl`
- Archiwa: `~/hexbane-archive/Races-2026-08-31/` (stara sztuka ras), `~/hexbane-archive/docs-2026-09-07/`
  (usunięte drzewa docs obu repo + `thoughts/`), `99_Archive/Projects/hexbane-notes-2026-08-31/`
  (stare notatki vaulta)
