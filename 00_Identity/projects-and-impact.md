---
type: identity
domain: [work, projects]
status: active
created: 2026-07-27
updated: 2026-07-27
tags: [career, projects, impact, interview-stories]
aliases: [projects-and-impact, projekty-i-wplyw]
---

# Projects and Impact

> Projekty, w których Filip rozpoznał problem, zaproponował kierunek i istotnie ukształtował rozwiązanie. Metryki wymagające uzupełnienia są zebrane w [[identity-gaps]].

## System wersjonowania procesów — Softeris

**Problem:** Procesy były ręcznie wyklikiwane, wielokrotnie testowane, a następnie odtwarzane u klienta wraz z ręcznym przesyłaniem plików.

**Wkład Filipa:** Samodzielna idea i implementacja mechanizmu wersjonowania procesów zapisanych w bazie. Proces przygotowany w środowisku deweloperskim można było jednym działaniem wdrożyć na platformę klienta przez mechanizm aktualizacji HTTP/SQL.

**Efekt:** Automatyzacja powtarzalnego, podatnego na błędy procesu wdrożeniowego. Rozwiązanie powstało na bardzo wczesnym etapie kariery.

## Framework cache — Picodi

**Problem:** Powtarzalny kod obsługi Redisa oraz trudna, podatna na błędy inwalidacja cache wpływały na wydajność i utrzymanie systemu.

**Wkład Filipa:** Zaprojektowanie fasady i interfejsu pozwalającego fragmentom logiki deklarować sposób cache’owania. Mechanizm wykrywał zmiany wpływające na dane i czyścił odpowiednie klucze.

**Efekt:** Ujednolicenie obsługi cache, ograniczenie duplikacji i bezpieczniejsza inwalidacja.

## Centralne zarządzanie konfiguracją i kolejkami — Volt

**Problem:** Współdzielone zmienne środowiskowe były powielane w wielu mikroserwisach, a ich zmiana wymagała aktualizacji w wielu miejscach. Brakowało też wspólnego modelu tworzenia kolejek.

**Wkład Filipa:** Zaproponowanie repozytoriów i spójnej konwencji nazewniczej oraz wdrożenie rozwiązania opartego o Terraform, Terragrunt i CI/CD do zarządzania konfiguracją i kolejkami SQS.

**Efekt:** Jedno źródło konfiguracji, mniej ręcznych zmian oraz bardziej powtarzalna infrastruktura.

## Voltus — kontrolowane wykonywanie komend przez Slack

**Problem:** Deweloperzy potrzebowali wykonywać komendy Symfony na kontenerach, ale bez bezpośredniego SSH i bez niekontrolowanego dostępu produkcyjnego.

**Wkład Filipa:** Zbudowanie w Pythonie bota Slack. Użytkownik wybierał komendę, parametry, klaster i serwis, po czym wymagane było zatwierdzenie przez osobę z odpowiednimi uprawnieniami.

**Efekt:** Szybsza samoobsługa deweloperów przy zachowaniu kontroli i audytowalnego procesu akceptacji.

## Platforma płatności stablecoinowych — Volt

**Problem:** Potrzeba uruchomienia płatności stablecoinami jako nowego obszaru zintegrowanego z istniejącym systemem Volt.

**Wkład Filipa:** Samodzielny research, proof of concept blockchain oraz projekt i implementacja mikroserwisu integrującego BVNK. Serwis został dopasowany do obecnego modelu danych i funkcji Volt, a jednocześnie przygotowany na przyszłe użycie własnej infrastruktury blockchain. Procesy płatnicze wykorzystują Temporal.

**Zakres:** Pay-in, pay-out, refund, settlement oraz transfery między portfelami.

**Efekt:** Nowy pion funkcjonalny wdrożony bez inwazyjnej przebudowy istniejącej architektury.

## Porażki i przełożone lekcje

### Niebezpieczne SQL na produkcji

Przypadkowe umieszczenie średnika przed WHERE doprowadziło do usunięcia całej zawartości tabeli ofert Valeo. Lekcja: bezpieczeństwo nie może zależeć wyłącznie od ostrożności; narzędzia i procesy powinny uniemożliwiać lub ograniczać ryzykowne operacje. Doświadczenie nauczyło też działania pod presją.

### Ewolucja systemu notyfikacji

System zaprojektowany początkowo dla jednego typu notyfikacji został po przekazaniu rozbudowany w sposób duplikujący odpowiedzialności i tabele. Lekcja: dobra architektura musi być zrozumiała dla kolejnych zespołów; handover, granice odpowiedzialności i dalsza opieka są częścią projektu.

### Wspólny user management dla sandbox i production

Jedna instancja serwisu obsługiwała oba tryby, aby użytkownicy i klienci nie byli duplikowani, a interfejs mógł przełączać środowisko. Z czasem rozwiązanie stało się problematyczne. Dziś Filip rozdzieliłby serwisy, a wspólne dane synchronizował podczas przepływu.

### Zbyt późna akceptacja Kubernetes

Filip długo preferował ECS, częściowo z powodu braku własnego doświadczenia z Kubernetes. Z perspektywy czasu widzi, że wcześniejsza migracja poprawiłaby doświadczenie deweloperów. Lekcja: znajomość obecnego rozwiązania nie powinna blokować uczciwej oceny alternatywy.

### Nadmierna centralizacja odpowiedzialności w infrastrukturze

Zespół infra przejmował zbyt wiele odpowiedzialności z obawy przed ryzykiem uszkodzeń. Prowadziło to do wąskich gardeł i wolniejszych wdrożeń. Lekcja: guardraile i bezpieczna samoobsługa skalują się lepiej niż kontrola.

## Wzorzec projektowy

Filip zwykle nie tworzy pojedynczego feature’u. Rozpoznaje powtarzalny problem i buduje mechanizm, platformę albo warstwę samoobsługową, która zwiększa skuteczność innych osób.

## Powiązane

- [[career-timeline]]
- [[strengths-and-risks]]
- [[leadership-and-culture]]
- [[tools-stack]]
