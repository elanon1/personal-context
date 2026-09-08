---
type: project
project: Hexbane
area: plans
status: implemented
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, implementation, progression, primary-spells, races]
---
# Przebudowa ras, primary i progresji — wykonanie

Użytkownik zatwierdził rozpoczęcie implementacji wszystkich zmian. Ta notatka zamyka propozycje [[2026-09-08-race-primary-progression-redesign]] i [[2026-09-08-xp-ranked-progression]] jako implementację roboczą w obu repozytoriach. Obowiązujące kontrakty: [[progression]], [[combat-stat-rules]], [[spell-system]], [[character-details]], [[rpcs]], [[combat-v2]], [[shared-types]], [[duel-v2-client]]. Bez commita i wdrożenia na działający serwer.

## Wykonany zakres

- Wspólny budżet 400+5/level, minimum10, bez rasowych grantów/limitów statystyk; softcap korzyści200, potem50%; darmowy pełny respec poza meczem. HP=(100+.5×softSTR)×cechaHP, mana=softINT; wszystkie rasy skalują moc z INT. Jeden profil obliczeń meczu i ekranu postaci, odpowiednik w kliencie.
- Human dodatkowy slot (start4, max7); Elf regeneracja many×1.2; Dark Elf obrażenia poison/hex×1.1; Shadow dodge .035/DEX do15%, casting−2; Gnome kosztmany×.85; Orc HP×1.05 i czas paralyze×.75. Stare bonusy szkolne i płaskie statystyki usunięte w migracji000005.
- Primary: graf DAG v1,6 tierów, checkpointy3/6, wybór ścieżek przez nowe RPC i klienta. Entitlement na levelach1/5/10/16/23/30, obydwa primary rozwijane niezależnie, bez kosztu MP. Zmiana wcześniejszego wyboru w UI zeruje późniejsze wybory; walidacja serwera wymaga legalnej spójnej ścieżki.
- Mirror:9mana,cast.5,recovery.4,okno1.5s,10% obrażeń zwracanych,100% ochrony przed jednym pakietem. W tierach2/4/5 +30pp zwrotu albo+1s okna; tier3 zwraca również statusy/DoT/hex. Tier6 Counterstroke: następny nie-primary cast−.1s przez2s po skutecznym zwrocie; albo Conservation: połowa rzeczywiście zapłaconej many przy niewykorzystanym wygaśnięciu.
- Arrow: dokładnie1 obrażenie, bez skalowania/treningu, zawsze zużywa lustro;5mana,cast.8,recovery.4. Tiery2/4/5:cast−.1 albo koszt−1. Tier3:po rozbiciu recovery−.1. Tier6:Opening następny nie-primary cast−.1 przez2s albo Rebate refund1many. Tempo nie stackuje. Odbicia zachowują ofensywny snapshot, mnożnik naliczany raz; małe odbite nie-Arrow mogą zaokrąglić się do0.
- Level30 przy6177XP;120wygrana/70porażka lub remis, bez daily bonusów. Około65meczów przy50%winrate;52samych wygranych/89porażek. Sloty7/11/16 (~5/10/20gier), Human zawsze+1. Posiadanie czarów i entitlement slotów pozostają niezależne.
- MP:5na levelach2/4/6,5na każdym8..16,2na każdym17..30:15/35/60/88 do7/11/16/30. Po capie500XP→5MP z resztą i overflow. Skill milestones25/50/75/100 każdego skilla→5MP tylko raz, bitmask persisted. Skille dalej do100, trening max5punktów/skilla/mecz i jedna próba na akcję, nie tick DoT; Arrow nie trenuje.
- Ranked: osobna partycja matchmakingu i przycisk klienta, wymagany tylko level30; serwer weryfikuje przy kolejce, tworzeniu meczu i join/rejoin. Brak gate'u skilli, MP, kolekcji. To dostęp do ranked, nie nowa drabinka MMR/rating.
- Baza:000004 konwertuje XP według części poziomu, resetuje statystyki do133/134/133+5×(level−1)do rozdania, zachowuje poziom, kolekcję, saldo i wcześniejsze sloty. Milestones istniejących skilli przyznane raz. Receipt `(character,match)` i rowlock zapewniają pojedyncze rozliczenie. Obie ludzkie nagrody zapisane przed broadcastem; przejściowy błąd ponawiany i blokuje zakończenie meczu. Lease buildu blokuje respec/primary/alokację podczas meczu.
- Klient: nowe primary RPC, edytor wyborów grafu, respec, studyXP, ranked gating, poprawiony fallback ras i preview statystyk; katalog `duel_v2.4`, protocol2.

## Weryfikacja

- `go test ./...` i końcowe `go test -race ./...`: PASS.
- `go vet ./...`: PASS. Serwer `git diff --check`: PASS. Pełny diff klienta zawiera wcześniejszą końcową spację w CreateCharacterCommandHandler.cs:42, poza zakresem tych zmian.
- `make build`: PASS, Linux plugin `build/backend.so`, pluginbuilder3.27.0.
- Klient rzeczywisty `dotnet build --no-restore`: PASS,0błędów/9istniejących ostrzeżeń. `dotnet run --project Tests/Progression/Progression.csproj`: PASS, budżet i preview.
- Osobny PostgreSQL17.6: `scripts/test_db_schema.sh` pełne up/down/up PASS. `TestRedesignPersistenceIntegration` i `TestRedesignMigrationIntegration` z `TEST_DB_URL` i `-race`: PASS. Sprawdzone8równoległych retry, pojedyncze XP/MP/milestone, późniejszy zakup/respec nie nadpisany receipt, blokady buildu, konwersja starej postaci, zachowanie czaru/slotów/balansu, rollback i jego guard.
- Symulacja seed42/1000: brak błędów zasobów,546rozstrzygnięć/454remisy(timeout), mediana143.1s. Tylko warunkowa próbka polityk/bazowych primary; nie dowodzi balansu ras. Szczegóły w [[spell-system]].

## Granice i dalsza walidacja

- Migracje sprawdzone wyłącznie na disposableDB. Do uruchomienia nowych binariów potrzebne000004/000005; należy wdrożyć zgodny klient i serwer razem.
- Down000004 przywraca zapisany snapshot starej progresji; celowo odmawia rollbacku, jeśli powstały nowe postacie bez snapshotu. W takim przypadku potrzebna jawna konwersja do starego modelu. Nie ma bezpiecznego automatycznego zachowania obu krzywych jednocześnie.
- Transakcyjny receipt chroni zapisane wyniki i retry w żywym procesie. Utrata procesu w trakcie awarii DB może zgubić jeszcze niezapisany wynik: trwały outbox całego meczu nie jest częścią tej implementacji.
- Bez testu interaktywnego nowych kontrolek w Godot i pełnego live meczu Nakama. Dużo timeoutów wymaga strojenia tempa i playtestu, zanim współczynniki zostaną uznane za finalny balans. Pierwsza paczka ma12opcjonalnych czarów, więc cel15+wyborów wymaga następnej paczki; MP można oszczędzać.
