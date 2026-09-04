---
type: project
project: Hexbane
domain: [projects, creative]
status: active
state: active
repo: https://github.com/elanon1/hexbane
created: 2026-08-31
updated: 2026-08-31
tags: [hexbane, gamedev, godot, csharp, nakama, go, kubernetes, ai-art]
aliases: [hexbane, hexbane-server]
---

# Hexbane — State

## Summary

**Hexbane** to gra mobilna/desktop 1v1: pojedynek magów w czasie rzeczywistym, bez ruchu — liczy się wybór zaklęć, timing rzucania (cast time, brak cooldownów) i medytacja (regeneracja many). Dwa repozytoria:

- **Klient** — `~/RiderProjects/hexbane` (`elanon1/hexbane`, branch `master`): Godot 4.5 + C# .NET 9, Nakama SDK 3.16, warstwy `Game → Application → Core`, DI + własny CQRS. → [[10_Projects/hexbane/architektura-klienta|architektura klienta]]
- **Serwer** — `~/GolandProjects/hexbane-server` (`elanon1/hexbane-server`, branch `main`): plugin Go do Nakama 3.27 (`backend.so`), Postgres 17, autorytatywne mecze fazowe 10 tick/s, 63 zaklęcia w YAML, system progresji RPG (35 ras, STR/INT/DEX, 3 skille, XP/poziomy/magic points). → [[10_Projects/hexbane/architektura-serwera|architektura serwera]]

Prod stoi na własnym klastrze k8s przez Helm + Argo CD: `hexbane.elanon.pl` (API) i `hexbane-console.elanon.pl`. → [[10_Projects/hexbane/infra-i-deploy|infra i deploy]]

Decyzja fundamentalna (26.06.2025): klient przepisany z GDScript na C# — łatwiej o ludzi, możliwa migracja poza Godota.

Pozostałe notatki projektu: [[10_Projects/hexbane/protokol-klient-serwer|protokół klient↔serwer]] · [[10_Projects/hexbane/zasady-gry|zasady gry (liczby)]] · [[10_Projects/hexbane/assety-i-pipeline|assety i pipeline AI-art]].

## Status

`active`, ale w **dwóch różnych tempach**:

- **Kod gry (klient + serwer)**: pętla rdzeniowa działa end-to-end — auth → tworzenie postaci → matchmaking (PvP / bot) → lobby z draftem zaklęć (35 s/tura, naprzemiennie) → 5-minutowy pojedynek → game over z XP/level-upem. Ostatnie wpisy devlogu: 31.12.2025; ostatnie commity serwera: bonusy żywiołowe ras + „fix effect phase”; klient: SDK 4.5.2, assety ras, ekran logowania.
- **Art / content**: 2026-06 masowa produkcja 35 ras skillem `race-maker`; **2026-08-31 cały roster wycofany** — będzie nowy od zera (patrz decisions log). `Resources/Races/` jest puste (zostały tylko `_tools/`), stara sztuka w archiwum `~/hexbane-archive/Races-2026-08-31/` (922 MB). Pełny rekonesans 2026-08-31: `thoughts/shared/research/2026-08-31-hexbane-client-server-recon.md` w repo klienta.

**Do zrobienia ręcznie (blokada klasyfikatora Claude Code, 2026-08-31):** plik `hexbane-server/db/migrations/000011_clear_races.up.sql` (`DELETE FROM characters; DELETE FROM races; ALTER TABLE characters ALTER COLUMN race_id DROP DEFAULT;`) + odpowiadający `down.sql` (re-insert 35 ras z `000010`). Kod i docs serwera już zakładają tę migrację. **Uwaga:** push tej migracji na `main` = Argo CD auto-sync + initContainer `migrate-custom` → **wyczyszczenie postaci na prodzie**.

**Kluczowe fakty przed planowaniem (stan na 2026-08-31):**

1. **Pojedynek nie renderuje ras.** W meczu gracz to generyczny rig `Skeleton2D` z `Resources/Chars/Concept1`. Podgląd rasy przy tworzeniu postaci (`RaceAnimationPreview`) czyta `Resources/Races/<id>/animation/frames.tres` — nowe rasy muszą dostarczać ten plik.
2. **4 z 11 typów efektów serwera nie mają handlera** (`stun`, `slowdown`, `absorb`, `mana_drain`) → ~18 z 63 zaklęć jest częściowo martwych (błąd tylko w logu).
3. **VFX po stronie klienta istnieje dla 9 zaklęć**, z czego 3 to id ze starego prototypu (`magic_sparkle`, `heal`, `flamestrike`), których serwer nie zna. 54 zaklęcia serwera nie mają żadnego VFX.
4. **Docs są za kodem**: `GUIDE-v2` (mecz + zaklęcia) opisuje archiwalny prototyp i błędne czasy; `docs/opcodes` mówi `GameCountdown=9`, kod ma `16`; `RPCs.md` nieaktualny; docs ras mówią „36”, migracja ma 35.
5. **Sekrety w repo**: serwer `.env.dist` (klucz OpenAI), `helm/hexbane/values.yaml` (GitHub PAT), klient `.env` (dev login/hasło, pakowane do builda jako `Content`).
6. Endless Story / AI-tworzenie postaci: kod OpenAI usunięty w working tree serwera; klient nadal woła `start_story` (na serwerze zakomentowane).
7. `deploy.sh` i `StartProgram` w csproj wskazują ścieżki Windows/WSL2 — nie z tego Maca.

## Decisions log

- **2026-08-31 — Ekran logowania przerobiony 1:1 pod referencję `reference/auth screen/full.png`.** Pływająca karta z poświatą zamiast pełnej ciemnej połowy, ornament + tytuł serif (Cinzel Decorative Regular + `FontVariation` spacing), pola z ikonami i podglądem hasła, 9-patchowy pomarańczowy przycisk, divider SERVER, dropdown z globusem, link w stopce. Nowy theme `_Themes/m_auth_card_theme.tres` (type variations) nakładany na `m_auth_theme`; assety w `Resources/Images/Auth/` wycięte z referencji (luminancja → alfa). **Why:** referencja powstała z tego samego tła, więc taniej było wyciąć z niej elementy niż generować od nowa; `DESIGN_SYSTEM.md` uzupełniony. Gotcha: tekstura przycisku wycięta z mocka miała wypalony napis — 9-patch rozciągał go pod prawdziwym tekstem (wyglądało jak zepsuty font).
- **2026-08-31 — Wycofany cały roster 35 ras; nowe rasy od zera.** Klient: `Resources/Races/*` (nigdy niezacommitowane, 922 MB) **przeniesione** do `~/hexbane-archive/Races-2026-08-31/` zamiast skasowane — jedyna forma odwracalności; zostało `_tools/`; sceny dev zaszyte na stare rasy (`Game/ScenesV3/Dev/VitraelPreview.*`, `Game/ScenesV3/RaceTest/*`) `git rm`. Serwer: hardcode `"glassvein"` (×4) zastąpiony stałą `race.DefaultRaceId` (`modules/race/types.go`, dziś `""` — `NewCharacter` bez rasy używa bazowych statów); `docs/progression/race.md` przepisany (sekcja „Roster” z checklistą dla nowej migracji); stare bible z `docs/client/races/` do archiwum. Migracja `000011_clear_races` **do dopisania ręcznie** (blokada klasyfikatora). **Why:** Filip: „powywalaj istniejące rasy, będziemy robić nowe”. Nic nie zacommitowane.
- **2026-08-31 — Rekonesans obu repo zapisany jako mapa techniczna + notatki w vaultcie.** Why: przed planowaniem kolejnych kroków potrzebny był jeden spójny obraz klienta, serwera, kontraktu między nimi i realnego stanu contentu, bo dokumentacja w repo rozjechała się z kodem w kilku miejscach (opcode 16 vs 9, czasy faz, katalog zaklęć).
- **2026-06 — Produkcja ras zautomatyzowana (skill `race-maker`, w pełni autonomiczny).** Solid chroma tło per rasa, zero malowanych efektów (VFX dodaje silnik), 6 animacji 3×3. Why: ręczne generowanie było wąskim gardłem; chroma zamiast alpha, bo ChatGPT nierzetelnie oddaje przezroczystość. (Szczegóły: pamięć Claude Code projektu hexbane.)
- **2026-06-18 — Klient: animacje ras jako sprite-sheety, nie rig kostny.** Rig z części (`RaceTest/TumbloamRig.tscn`) przetestowany i odłożony; pipeline docelowo daje `SpriteFrames`. Why: rig wymagał generowania kompletnych części ciała per rasa; sheety skalują się na 35+ ras.
- **2026-01-30 — Serwer: docs v2 (`DOCUMENTATION-INDEX`, `QUICKSTART-v2`, `API-REFERENCE-v2`, `GUIDE-v2`).** Od tego czasu kod poszedł dalej (63 zaklęcia, żywioły ras), docs nie.
- **2025-08-12 — Serwer: model „mecz prowadzony przez jednego gracza” zamiast czystych RPC dla tworzenia postaci (Endless Story).** Why: zarządzanie sesją/pamięcią przez Nakama match zamiast trzymać stan w RPC.
- **2025-07-29 — `PlayerState` pod mutexem, atomowy `TryCastSpell`.** Why: prawie równoczesne casty psuły animacje i zaklęcia (race condition).
- **2025-07-27 — Efekty: brak stackowania tego samego zaklęcia (recast odświeża czas).** Dziś: kolejka per mecz z kluczem `spellID/effectType` — różne zaklęcia tego samego typu stackują się obok siebie.
- **2025-06-26 — Klient przepisany z GDScript na C#.** Why: GDScript to nisza (trudno o ludzi), C# daje perspektywę migracji na inne silniki.

## Open questions

- Który zestaw zaklęć jest kanoniczny dla produkcji ikon/VFX/SFX: 63 z YAML serwera czy 19 folderów w `Resources/Spells/` (głównie stare id)?
- Nowy roster ras: ile, jaki styl, jaka rasa domyślna (`race.DefaultRaceId` + default kolumny `race_id`), jakie żywioły (muszą pasować do szkół serwera: air/earth/fire/ice/lightning/mind/neutral/toxic/water) i odporności (klucze = żywe id zaklęć)?
- Czy pojedynek ma renderować rasy — i którą drogą: `SpriteFrames` (`frames.tres`) czy rig?
- `stun`/`slowdown`/`absorb`/`mana_drain`: dopisać handlery czy wyciąć z katalogu?
- Endless Story: porzucone czy przepisywane bez OpenAI?
- Element ras (`docs/progression/race.md`) odwołuje się do szkół, których serwerowe zaklęcia nie mają → bonusy żywiołowe w praktyce martwe poza pokrywającymi się nazwami. Które szkoły są docelowe?
- Sekrety w repo — do rotacji i usunięcia z historii (nie zrobione, tylko odnotowane).
- Deploy z Maca: brak działającej ścieżki (skrypty pod Windows/WSL2).

## Links

- [[NOW]] · [[tools-stack]] (Godot, C#, Go, Nakama, k8s, Argo CD)
- [[10_Projects/cluster-agent/_state|Cluster Agent]] — ten sam klaster k8s, na którym stoi hexbane (Argo CD w `elanonix/argocd`)
- Repo klient: `github.com/elanon1/hexbane` · serwer: `github.com/elanon1/hexbane-server`
- Obraz: `ghcr.io/elanon1/hexbane-server:latest` · prod: `https://hexbane.elanon.pl`
- Mapa techniczna (pełna, z odnośnikami do linii): `hexbane/thoughts/shared/research/2026-08-31-hexbane-client-server-recon.md`
- Docs w repo: klient `docs/DESIGN_SYSTEM.md`, `docs/opcodes/`; serwer `docs/DOCUMENTATION-INDEX.md`, `docs/progression/*`, `docs/devlog.md`
