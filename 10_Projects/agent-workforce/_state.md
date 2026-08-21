---
type: project
project: Agent Workforce
domain: [projects, creative]
status: active
state: active
repo: https://github.com/elanonix/argocd
created: 2026-07-27
updated: 2026-08-21
tags: [agents, n8n, automation, side-income]
aliases: [agent-workforce, agenci-pracownicy, zaloga-agentow]
---

# Agent Workforce — State

## Summary

Rozszerzenie osobistych agentów n8n z **asystentów** (oszczędzają czas) w **pracowników** (wytwarzają artefakty i docelowo przynoszą pieniądze). Fundament już stoi: `Personal Assistant (Master)` na Telegramie z pamięcią w Postgresie, plus bezstanowe sub-agenty `budget_agent` i `obsidian_agent`. Ten projekt dokłada do tego załogę produkcyjną: Shipper → Publicist → Scout → Analyst.

Cel finansowy to **pocket money, bez horyzontu czasowego**. Realna ścieżka pieniądza nie prowadzi przez bezpośrednią sprzedaż narzędzi, tylko: artefakty → widoczność → przychodzące zlecenia usługowe. Uzgodnione, że to perspektywa 6–12 miesięcy i że to jest akceptowalne.

## Status

`planning`. Nic z załogi jeszcze nie istnieje. Zrobiona jest warstwa pamięci, na której cała reszta się oprze (patrz wpis z 2026-07-27 poniżej). Następny krok merytoryczny: **kontrakt zadania dla Shippera** — co dokładnie dostaje na wejściu, żeby wyprodukować PR ocenialny w 5 minut.

### Załoga — kolejność budowy

1. **Shipper** — rdzeń. Zadanie → headless Claude Code w repo → kod, testy, README, release notes → PR do zatwierdzenia. Bez niego reszta nie ma o czym pisać ani co mierzyć.
2. **Publicist** — z każdego zmergowanego PR-a robi publiczny artefakt (post, changelog, demo, README). Żywi się wyłącznie realną pracą, nigdy nie wymyśla treści.
3. **Scout** — skanuje issues, rejestry MCP i wątki gamedev po luki narzędziowe. Max 3 kandydatury tygodniowo, każda z „dlaczego teraz" i szacunkiem pracy.
4. **Analyst** — gwiazdki, pobrania, ruch; wnioski wracają do Scouta.

### Ograniczenia (wymagania, nie preferencje)

- **Autonomia = kolejka do zatwierdzenia.** Agent doprowadza rzecz do końca i czeka na jedno „ok". Nigdy nie publikuje sam.
- **Budżet recenzji: ~30 min dziennie** → realnie 2–4 decyzje. Selektywność ważniejsza od przepustowości; żadnych półproduktów w kolejce.
- **Odporność na zaniedbanie.** Główne ryzyko to nie porażka, a ciche obumarcie w drugim miesiącu. Agent milczy, gdy nie ma czego pokazać; pozycje w kolejce wygasają, zamiast kumulować się w wyrzut sumienia.
- **Artefakty bez dyżurów** — paczki typu wydaj-i-zapomnij (MCP, CLI, skille), nie hostowany SaaS.

## Decisions log

- **2026-07-27 — Kolejką zatwierdzeń jest lista PR-ów na GitHubie, nie własny mechanizm w n8n.** Skoro Shipper kończy PR-em, to „ok" = merge z telefonu. Nie budujemy stanu, wygasania ani UI kolejki — nietknięty PR po prostu leży i nikomu nie szkodzi, co samo załatwia wymóg odporności na zaniedbanie.

- **2026-07-27 — Substrat Shippera: najpierw Mac (PoC), potem GitHub Actions.** Dlatego substrat musi być wymienialny od pierwszego dnia: n8n zna wyłącznie kontrakt „specyfikacja zadania → PR", a runner to cienki adapter (dziś SSH na Maca, jutro `workflow_dispatch`). Bez tego rozdzielenia migracja za dwa miesiące będzie przepisywaniem, nie podmianą. Świadomie przyjęte, że na Macu „agent pracuje, gdy śpisz" nie jest prawdą — PoC ma dowieść czegoś innego.

- **2026-07-27 — Kryterium sukcesu PoC to czas recenzji, nie to, czy skrypt się odpalił.** Prawdziwe ryzyko przy 30 minutach dziennie: jeśli PR wymaga 40 minut czytania, model się zawala niezależnie od infrastruktury i dostajesz drugą pracę zamiast side money. Stąd wymóg: PR-y małe, jednotematyczne, z dowodem przetestowania.

- **2026-07-27 — Shipper budowany niszowo-obojętnie.** Jego kontrakt to „zadanie → PR", bez wiedzy o gamedevie czy MCP. Nisza siedzi wyłącznie w konfiguracji Scouta (co skanuje) i w tonie Publicisty. Dzięki temu mechanizm nr 2 na zupełnie inny temat kosztuje wymianę tych dwóch, a nie budowę od zera.

- **2026-07-27 — Scout przesunięty z pierwszego na trzecie miejsce.** Pierwotny pomysł zakładał agenta szukającego możliwości dochodu jako punkt startowy. Odrzucone: sam skaut okazji produkuje listy, których nikt nie realizuje, bo wąskim gardłem nie jest brak pomysłów, tylko dystrybucja i dowód kompetencji. Dziś backlog własnych pomysłów jest pełny (`SpellForge`, generatory asetów i muzyki czekają nieopakowane) — Scout ma sens dopiero, gdy wyschnie.

- **2026-07-27 — Odrzucona warstwa pośrednia `business_agent` (router nad sub-agentami).** Przy 4–5 narzędziach router nic nie kupuje, a kosztuje: dodatkowy szczebel LLM rozmywa dosłowne przekazywanie odpowiedzi i stan pending, dokłada latencję i drugi prompt do utrzymania. **Warunek powrotu do pomysłu:** master ma >5–6 narzędzi biznesowych i zaczyna wybierać złe, albo biznes potrzebuje innej osobowości/modelu niż część osobista. Dodanie warstwy będzie wtedy tanie — grupujesz istniejące workflow i zmieniasz jedną linię w promptcie.

- **2026-07-27 — Odrzucony osobny bot biznesowy na Telegramie.** Daje najczystszą izolację, ale cofa decyzję z 2026-07-26 („wszystko za jedną rozmową"): dwa czaty, utrata wiadomości wielointencyjnych, zduplikowane STT i obsługa błędów, trzeci token. Izolację, o którą naprawdę chodzi, załatwia **zakres pamięci**, nie osobny byt.

- **2026-07-27 — Prawdziwa granica przebiega między pracą synchroniczną a asynchroniczną, nie między „personal" a „business".** Budget to pytanie → odpowiedź w 5 sekund. Shipper to zadanie na 40 minut, którego wynikiem jest PR, a nie zdanie w czacie. Shipper **nie może** być `toolWorkflow` na żadnym szczeblu — wysypie się na timeoucie. W rozmowie zostają cienkie tools („zleć", „co się dzieje", „odrzuć"); ciężka praca biegnie poza rozmową i melduje się powiadomieniem z linkiem.

- **2026-07-27 — Pamięć rozdzielona na dwie warstwy (zrobione).** Historia rozmowy → `Postgres Chat Memory` w bazie `agent_db`. Trwałe fakty → tabela `facts` przez tools `recall_facts` / `remember_fact`. Powód rozdzielenia: historia to przesuwne okno i się starzeje; gdyby fakty w niej siedziały, wyparowałyby razem z nią. Kolumna `scope` przygotowana pod `personal` / `business`, żeby Publicist nie mógł wciągnąć prywatnego szczegółu do publicznego posta.

- **2026-07-27 — Nisza: narzędzia dla developerów (extensiony, MCP, skille, CLI) z użytkownikami wśród gamedevów.** Wybrane zamiast usług i self-hostingu, bo są produktowe: agent może je pchać, gdy Filip śpi, a usługi wymagałyby jego kalendarza i synchronicznego kontaktu z klientem przy pracy na etacie. Punkt startu: **zero publicznej powierzchni** — brak portfolio, treści i audience.

## Open questions

- Kontrakt zadania dla Shippera — co dokładnie na wejściu, żeby PR dało się ocenić w 5 minut? To sedno; reszta to hydraulika.
- Pierwszy artefakt do przetestowania całości. Kandydat: najcieńszy sensowny MCP serwer nad jednym z istniejących generatorów — mały, kończy się paczką, od razu daje Publicistowi materiał.
- Kiedy i jak egzekwować izolację pamięci biznesowej — osobna rola w Postgresie czy wystarczy `scope` + dyscyplina promptu? Rozstrzygnąć, zanim powstanie Publicist, nie później.
- Rotacja sekretów: `ENCRYPTION_KEY` n8n i hasło do Postgresa leżą jawnie w repo `argocd`. Niezwiązane z tym projektem, ale dotyka tej samej infrastruktury.

## Links

- [[NOW]] — bieżący kontekst
- [[profile]] — kim jest Filip, preferencje komunikacyjne
- [[tools-stack]] — n8n, Kubernetes, Claude Code
- Repo GitOps: `github.com/elanonix/argocd` (n8n, Postgres, Obsidian, STT)
- `Personal Assistant (Master)` — n8n `eiuCVFO2GySjtEUB`

## 2026-08-21 — Nocna Fabryka: zwiad i wycena działają na produkcji

Filip zlecił (goal): system, w którym agenty nocą wymyślają i wyceniają pomysły dochodowe,
a on rano dostaje karty do decyzji (zarys, wdrożenie, godziny człowieka, prognoza PLN, zwrot).
Pełny plan: artefakt „Nocna Fabryka" (link w repo `contexts/2026-08-21-nocna-fabryka.md`).

**Zbudowane i opublikowane (tydzień 1 roadmapy):**
- Tabele `ideas` / `experiments` / `revenue` w agent_db (lejek raw→scored→approved→building→launched→earning; SQL też w gitops jako initdb `02-venture-factory.sql`).
- `Venture Scout (Nightly)` — 23:00 PL: HN Ask/front + Lemmy (asklemmy, nostupidquestions) + Stack Exchange Lifehacks → agent wybiera 3–8 ludzkich problemów → ideas(raw). Gate: dyrektywa `pause_factory`.
- `Venture Analyst (Nightly)` — 00:30 PL: do 6 pomysłów/noc, karta pomysłu w PLN (P10/P50/P90, kill criteria, zakaz zmyślania liczb rynkowych) → scored/rejected.
- `Venture Daily` — 07:30 PL: deterministyczny digest na Telegram (max 4 karty, eksperymenty, przychód 7 dni).
- Master: toole `decide_idea` („bierz #14" / „ubij #12") i `list_ideas`.
- Test na żywo: Scout znalazł 6 sygnałów, Analyst odrzucił wszystkie z konkretnymi powodami — selektywność zgodna z budżetem recenzji 30 min/dzień.

### Decyzje (2026-08-21)

- **Scout i Analyst jednak budowane teraz, przed Shipperem** — świadoma zmiana decyzji z 2026-07-27 („Scout trzeci"). Powód: Filip wprost zlecił autonomiczne generowanie pomysłów pod dochód pasywny; tamta decyzja zakładała inny cel (dowód kompetencji przez artefakty z własnego backlogu). Reszta ustaleń z 27.07 (kolejka = PR-y, 30 min/dzień, Shipper async i niszowo-obojętny) pozostaje w mocy i jest wpisana w plan fabryki.
- **Scenariusz inwestycyjny B**: ~1000 zł/mies. (API + testy walidacyjne ads 150–200 zł/pomysł). Wybrany zamiast bootstrap (A) i agresywnego (C).
- **Digest fabryki osobno** (07:30, bot Personal Agent), nie w Garmin Morning Briefing.
- **LLM nocny: CLIProxyAPI / claude-sonnet-5** — koszt nocnych runów ~0.
- **Czysty zwiad zamiast pomysłów z półki** — estate-digest/arXiv/Nakama odrzucone jako pierwszy wsad („potrzebujemy czegoś świeżego").
- **Źródła = ludzkie problemy, nie biznes-talk** — subreddity r/SaaS itp. odrzucone przez Filipa; docelowo codzienne frustracje ludzi. Wymuszone też technicznie: Reddit blokuje anonimowy JSON (403 nawet z domowego IP) → Lemmy + Stack Exchange Lifehacks; powrót Reddita tylko przez OAuth.

### Następne kroki

1. Obserwacja 2–3 nocy (jakość kart, czy Lemmy/SE konkurują z HN w wyborach Scouta).
2. Shipper (tydzień 3): kontrakt „karta approved → PR z PoC", SSH na Maca, `Propose Change`; przy „bierz" wiersz w `experiments`.
3. Publicist (tydzień 4) + pierwszy launch (Paddle/Lemon Squeezy — KYC Filipa). Przedtem: scope `business` w facts.

Uwaga operacyjna: harmonogramy instancji liczą w UTC (triggery = PL−2, latem) — po zmianie czasu w październiku przesunąć o godzinę.
