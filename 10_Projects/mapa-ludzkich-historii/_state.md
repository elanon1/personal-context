---
type: project
domain: [personal-life, creative]
status: active
project: Mapa Ludzkich Historii
repo:
state: planning
created: 2026-07-27
updated: 2026-07-27
tags: [project, product, memorial, maps, startup]
aliases: [herelived, Mapa Ludzkich Historii, archiwum ludzkości, atlas of lives]
---

# Mapa Ludzkich Historii — projekt

> **Jedno zdanie:** Wikipedia zwykłych ludzi, nałożona na mapę świata.
> Status: koncepcja + MVP na papierze (lipiec 2026). Następny krok → walidacja bez kodu (pkt 9).

## Powiązania
Konkurencja / referencje: [[StoryWorth]] · [[Remento]] · [[Tell Mel]] · [[Ślad Pamięci]] · [[MyWay]] · [[Find a Grave]] · [[BillionGraves]] · [[HereAfter]] (przestroga).
Tematy: [[RODO]] · [[DSA]] · [[dobra osobiste (KC)]] · [[kult pamięci osoby zmarłej]] · [[voice-first UX dla seniorów]] · [[zasada 3-2-1 backup]] · [[strategia zimnego startu]].

---

## 1. Wizja
Każdy człowiek nosi w sobie historię wartą zapamiętania — nie tylko sławni. Serwis pozwala zwykłym ludziom nagrać i spisać swoje życie za życia, a po śmierci ich historia staje się częścią publicznej, przypiętej do miejsc mapy ludzkich losów. Podróżując przez wieś czy miasto, możesz otworzyć aplikację i dowiedzieć się, kto mieszkał w mijanym domu, kim był i co przeżył.

Jedno zdanie: **Wikipedia zwykłych ludzi, nałożona na mapę świata.**

## 2. Problem
Miliardy ludzi odchodzą zapamiętane tylko przez najbliższych, a po jednym–dwóch pokoleniach — przez nikogo. Istniejące narzędzia ([[StoryWorth]], Lalo, strony memorialne) zamykają historie w prywatnych skrzynkach dla rodziny. Nie istnieje miejsce, gdzie historia zwykłego człowieka jest publicznie dostępna i powiązana z miejscem, w którym żył.

## 3. Wyróżnik (dlaczego to nie jest kolejny StoryWorth)
1. **Mapa, nie archiwum** — historie przypięte do domów, wsi, ulic, cmentarzy. Eksplorujesz życia ludzi tak, jak eksploruje się zabytki.
2. **Publiczność jako domyślny cel** — historia ma przetrwać i być dostępna dla świata, nie tylko dla wnuków.
3. **Suwak prywatności z mechanizmem "po śmierci"** — każdy fragment historii może być: prywatny / dla rodziny / publiczny teraz / publiczny po mojej śmierci. Ostatnia opcja zachęca do szczerości.
4. **Wartość dla biernych użytkowników** — podróżnik, który nic nie nagrywa, też ma po co wejść. To rozwiązuje problem "pustego serwisu".

## 4. Kluczowe mechanizmy
### Cykl życia historii
Osoba nagrywa/spisuje historię za życia (audio, tekst, zdjęcia) → ustawia poziomy prywatności per fragment → po zweryfikowanej śmierci: odblokowują się treści "po śmierci" oraz dokładna lokalizacja (za życia lokalizacja tylko z precyzją do miejscowości/dzielnicy — ochrona przed stalkingiem).

### Weryfikacja śmierci (hybryda)
- **Powiernik** (1–2 zaufane osoby wskazane przy rejestracji) zgłasza zgon + skan aktu zgonu, weryfikacja ręczna.
- **Dead man's switch** jako zabezpieczenie: cykliczne "potwierdź, że żyjesz" (mail/SMS); brak reakcji uruchamia wieloetapową procedurę z ostrzeżeniami i kontaktem z powiernikiem.
- Zawsze wieloetapowe potwierdzenie przed publikacją — fałszywe "uśmiercenie" żywego użytkownika to katastrofa nie do cofnięcia.
- Efekt uboczny: coroczny kontakt = naturalna okazja do retencji ("dodasz nową historię?").

### Walka z pustą kartką
- Cotygodniowe pytania-podpowiedzi (model [[StoryWorth]]): "Jak wyglądało Twoje pierwsze mieszkanie?", "Kogo z sąsiadów pamiętasz najlepiej?"
- Nagrywanie głosem jako główny format — starsi ludzie chętniej mówią, niż piszą ([[voice-first UX dla seniorów]]).
- Historie miejsc, nie tylko ludzi: dom, sklep, młyn, których już nie ma.

## 5. Zakres MVP
### Wchodzi (wersja 1)
- Profil osoby: tekst + audio + zdjęcia, pytania-podpowiedzi co tydzień.
- Suwak prywatności per fragment (prywatne / rodzina / publiczne / publiczne po śmierci).
- Mapa publicznych historii z precyzją do miejscowości; widok "w pobliżu".
- Weryfikacja śmierci: powiernik + skan aktu zgonu, ręczna moderacja.
- Zgłaszanie treści (notice-and-action) + moderacja przed publikacją treści publicznych (AI + wyrywkowo człowiek).
- Karta podarunkowa ("podaruj babci jej historię").

### Nie wchodzi (świadomie później)
- Dead man's switch automatyczny (v2 — na start wystarczy powiernik).
- Kody QR na nagrobki i tabliczki na domy (v2 — wymaga partnerów kamieniarskich/pogrzebowych).
- Integracja z rejestrem PESEL (v3 — bariera prawna).
- Awatary AI / "rozmowa ze zmarłym" (świadomie NIE — [[HereAfter]] padł, teren etycznie grząski, psuje zaufanie).
- Wideo (koszty storage; audio + zdjęcia wystarczą na start).

## 6. Model zarabiania
| Źródło | Opis | Kiedy |
|---|---|---|
| "Cyfrowy grobowiec" | Jednorazowa opłata (300–500 zł) za wieczyste utrzymanie historii; część kwoty na fundusz utrzymania | MVP |
| Karta podarunkowa | Dzieci/wnuki kupują usługę seniorowi (Dzień Babci, urodziny, święta) — główny kanał sprzedaży | MVP |
| Drukowana książka | Fizyczna pamiątka z historii (druk na żądanie) | v2 |
| Tabliczki QR | Nagrobki, domy — produkt fizyczny, kanał: zakłady kamieniarskie i pogrzebowe | v2 |
| B2B | Hospicja, domy opieki, gminy, muzea regionalne (lokalne archiwa historii mówionej) | v3 |

Zasada: **unikać subskrypcji jako głównego modelu** — ludzie nie chcą płacić miesięcznie za coś, co "zadziała" po ich śmierci (lekcja z upadku [[HereAfter]]). Darmowy poziom podstawowy musi istnieć, żeby mapa się zapełniała.

## 7. Ryzyka prawne i jak je ograniczamy
⚠️ *Do przejścia z kancelarią przed startem. Regulamin nie chroni przed roszczeniami osób trzecich, które go nie akceptowały.*
- **Treści o osobach trzecich** (zniesławienie, [[dobra osobiste (KC)]] art. 23–24 KC, [[RODO]]): zasada produktowa "opowiadaj o sobie, nie oceniaj innych z nazwiska", moderacja przed publikacją, opcja anonimizacji nazwisk, domyślne odroczenie wrażliwych fragmentów do śmierci osób opisywanych.
- **Obowiązki platformy ([[DSA]])**: mechanizm zgłaszania i usuwania treści, reagowanie na zgłoszenia.
- **[[kult pamięci osoby zmarłej]]** (KC): RODO nie chroni zmarłych, ale rodzina może pozwać za szkalowanie — moderacja obejmuje też historie o zmarłych.
- **Lokalizacja żyjących**: dokładny adres ujawniany dopiero po śmierci; za życia precyzja do miejscowości.
- **Fałszywe zgłoszenie zgonu**: wieloetapowa weryfikacja, akt zgonu, okres karencji z powiadomieniem użytkownika wszystkimi kanałami.

## 8. Metryki sukcesu MVP
- Liczba rozpoczętych historii i **odsetek historii z ≥5 fragmentami** (czy ludzie wracają nagrywać — kluczowa metryka).
- Odsetek fragmentów oznaczonych "publiczne" lub "publiczne po śmierci" (czy wizja publicznego archiwum działa).
- Sprzedaż kart podarunkowych vs. zakupy własne.
- Ruch na mapie od użytkowników, którzy nic nie nagrali (wartość dla biernych).

## 9. Najbliższe kroki
1. **Test hipotezy bez kodu**: 10–15 rozmów z seniorami (czy chcą opowiadać? co ich blokuje?) i z ich dziećmi (czy kupią to jako prezent?).
2. **Pretotyp mapy**: ręcznie zebrać 20–30 historii z jednej miejscowości (koło historyczne, biblioteka gminna) i pokazać ludziom — czy chcą to przeglądać?
3. Konsultacja prawna ([[RODO]], [[DSA]], dobra osobiste, procedura zgonu).
4. Dopiero potem: budowa MVP.

## 10. Analiza konkurencji (lipiec 2026)
Rynek dzieli się na trzy grupy — **nikt nie łączy wszystkich trzech elementów naszego pomysłu**.

### Grupa 1: Nagrywanie historii za życia (prywatne, dla rodziny)
| Gracz | Model | Uwagi |
|---|---|---|
| [[StoryWorth]] | Cotygodniowe pytania mailem → książka (~99 USD/rok) | Lider: 35+ mln historii, 1 mln wydrukowanych książek, 12 lat na rynku |
| [[Remento]] | Voice-first, AI transkrypcja, QR w książce do nagrań głosu | Inwestycja Marka Cubana (Shark Tank 2025, 300 tys. USD); rosnące tempo |
| [[Tell Mel]] | AI-biograf dzwoni do seniora i prowadzi rozmowę telefoniczną | Zero technologii po stronie seniora — ważny wzorzec UX |
| Storii, Meminto, HeritageWhisper, Willow i in. | Warianty: telefon, wspólne nagrywanie, tańsze plany | Rynek rozdrobniony, niska bariera wejścia |

**Wspólna cecha i słabość**: wszystko kończy się prywatną pamiątką dla rodziny. Żaden nie buduje publicznego archiwum ani warstwy miejsc.

### Grupa 2: Polska — upamiętnianie po śmierci
| Gracz | Model | Uwagi |
|---|---|---|
| [[Ślad Pamięci]] | Profile grobów, wspomnienia, wirtualne znicze, tabliczki QR, poziomy prywatności; od 79 zł/rok + opcja jednorazowa "Na Zawsze" | **Najbliższy konkurent w PL** — ma QR i model wieczysty, ale startuje od grobu, nie od żywego człowieka |
| [[MyWay]] | Cyfrowe wspomnienia przy grobach (QR na zniczu) + zdalne usługi cmentarne (znicz, kwiaty, sprzątanie) | Model oparty na usługach; rynek zniczy w PL ~1 mld zł/rok |
| OgrodyWspomnien.pl, Grobonet, wyszukiwarki grobów | Wirtualne cmentarze, księgi pamiątkowe, wyszukiwarki pochówków | Starsza generacja produktów, słaby UX, ale zajęte pozycje w SEO |

**Wspólna cecha i słabość**: zaczynają od śmierci i od grobu. Nikt nie zbiera historii od żywych, własnym głosem.

### Grupa 3: Mapa + zwykli ludzie (skala, ale bez opowieści)
| Gracz | Model | Uwagi |
|---|---|---|
| [[Find a Grave]] (Ancestry) | Społecznościowe wspomnienia przy grobach | 265+ mln wspomnień od 1995 r. — dowód, że ludzie masowo dokumentują życie obcych za darmo |
| [[BillionGraves]] | Zdjęcia nagrobków z GPS, baza genealogiczna, AI "Cemetery Intelligence" | Każdy rekord = zdjęcie + współrzędne; wolontariusze fotografują całe cmentarze |

**Wspólna cecha i słabość**: mapują groby i daty, nie historie życia. Narzędzia genealogiczne, nie opowieści.

### Wnioski strategiczne
1. **Luka jest precyzyjna**: historia opowiedziana za życia własnym głosem → publiczna → przypięta do miejsc życia (dom, wieś, ulica), nie tylko do grobu. Tego nie robi nikt.
2. **Główne zagrożenie**: [[Ślad Pamięci]] lub [[MyWay]] dodają moduł "nagraj za życia". Tempo ma znaczenie.
3. **Fosa**: suwak prywatności "publiczne po śmierci" + mapa całego życia (nie cmentarza) + wartość dla biernych eksploratorów.
4. **Lekcje z rynku**: voice-first wygrywa u seniorów (Remento vs StoryWorth); telefon bez aplikacji obniża barierę (Tell Mel, Storii); wolontariusze zapełnią mapę, jeśli da się im misję (BillionGraves); upadek [[HereAfter]] AI = przestroga przed subskrypcją i awatarami AI.

## 11. Nazwa i domeny
Wolne domeny .com (zweryfikowane w rejestrze Verisign, 10.07.2026 — dostępność może się zmienić w każdej chwili, koszt rejestracji ~50–60 zł/rok):
| Domena | Charakter | Uwagi |
|---|---|---|
| **herelived.com** | ⭐ rekomendacja | Krótka; "tu żył..." — dokładnie moment odkrycia z podróży; echo kamieni Stolpersteine ("Hier wohnte") |
| **atlasoflives.com** | wizja globalna | "Atlas ludzkich losów" — poważny ton, dobrze skaluje się międzynarodowo |
| **mapoflives.com** | dosłowna | Prostsza wersja powyższej |
| **everyonelived.com** | misyjna | "Każdy zasługuje na pamięć" |
| **herewaslife.com** | poetycka | "Tu było życie" — pasuje też do historii miejsc, które zniknęły |

Do zrobienia przy wyborze:
- Zarejestrować od razu 2–3 warianty (wolna domena może zniknąć z dnia na dzień).
- Dokupić odpowiednik **.pl** dla startu na rynku polskim.
- Sprawdzić kolizje znaków towarowych (UPRP / EUIPO) przed inwestycją w markę.
- Sprawdzić dostępność nazw na social mediach (Instagram, TikTok, YouTube).

## 12. Infrastruktura i koszty
### Skala danych
Audio kompresowane (Opus/AAC, jakość mowy): ~0,5 MB/min. Pełna historia życia (3–5 h nagrań + kilkadziesiąt zdjęć) ≈ **200–300 MB/osobę**. 1000 użytkowników ≈ 250 GB → przy cenach object storage (~25 zł/TB/mies.) koszt liczony w pojedynczych złotych miesięcznie. Opłata "cyfrowy grobowiec" (300–500 zł) pokrywa storage na dekady; realny koszt "wieczystości" to utrzymanie organizacji, nie bajty.

### Architektura przechowywania (RODO-first)
| Warstwa | Rozwiązanie | Uwagi |
|---|---|---|
| Pliki (audio, zdjęcia) | Object storage w UE: Hetzner (DE), OVH (DC w Warszawie), Cloudflare R2, Backblaze B2 | R2: darmowy transfer wychodzący — istotne przy masowym odsłuchu z mapy; B2: najtaniej (~6 USD/TB) |
| Baza (profile, lokalizacje, uprawnienia, suwak prywatności) | PostgreSQL | Na start na tym samym VPS, potem wersja zarządzana |
| Kopie archiwalne | Zimny storage (np. Glacier Deep Archive ~1 USD/TB/mies.) u **innego** dostawcy | [[zasada 3-2-1 backup]]: 3 kopie, 2 technologie, 1 inna lokalizacja — fundament obietnicy "na zawsze" |
| Mapa | MapLibre + OpenStreetMap | Za darmo; unikać Google Maps (koszty rosną z ruchem) |

### Budżet miesięczny MVP
- VPS (aplikacja + baza): 100–200 zł (Hetzner/OVH)
- Storage + CDN: 0–50 zł na start
- Transkrypcja audio (Whisper API): ~1–2 USD / pełna historia — wliczyć w cenę pakietu
- Domeny, mail: ~30 zł
- **Razem: 200–500 zł/mies.** przy pierwszych setkach użytkowników

Prawdziwe koszty: czas developera, moderacja, prawnik, marketing — infrastruktura to margines.

### Plan na śmierć firmy (przewaga zaufania)
Publiczna deklaracja: w razie zamknięcia serwisu (a) rodziny dostają pełny eksport danych, (b) publiczne archiwum przechodzi do partnera archiwalnego (biblioteka cyfrowa, archiwum społeczne, fundacja). Lekcja z upadku [[HereAfter]] AI — patrz pkt 10.4.

## 13. Strategia zimnego startu: lokalni sławni
Problem pustej mapy rozwiązujemy pierwszą warstwą treści: **postacie sławne lokalnie, nie globalnie** (przedwojenny doktor, nauczycielka-legenda, ostatni młynarz, powstaniec). Globalne sławy (Chopin, Skłodowska) — świadomie NIE: Wikipedia robi to lepiej, a marka dryfowałaby w stronę encyklopedii sławnych, wbrew wizji "Wikipedii zwykłych ludzi".

**Źródła treści (legalne, tanie):**
- Wikidata/Wikipedia: współrzędne tablic pamiątkowych, pomników, miejsc urodzenia + życiorysy na wolnych licencjach
- Wikimedia Commons: zdjęcia (CC — pilnować licencji)
- Izby pamięci, towarzystwa i koła historyczne, biblioteki gminne: biogramy + gotowi partnerzy i ambasadorzy
- Fizyczne tablice "Tu mieszkał..." — digitalizacja istniejącej warstwy pamięci

**Prawnie:** RODO nie chroni zmarłych; fakty biograficzne osób publicznych dozwolone; uwaga na licencje zdjęć i rzetelność przy postaciach kontrowersyjnych (kult pamięci → roszczenia rodziny).

**Mechanika konwersji:** lokalny sławny = przynęta ("zobacz historię doktora z Twojego miasta") + wzorzec dobrej historii (obniża barierę pustej kartki) + trigger: "mój dziadek miał ciekawsze życie".

**Plan startu:** 2–3 miasteczka wypełnione gęsto (30–50 postaci + historie miejsc), partnerstwo z kołem historycznym i biblioteką, test: czy mieszkańcy zaczną dodawać swoich bliskich. Spójne z pretotypem z pkt 9.

---
*Dokument roboczy — kolejne kroki: rozmowy walidacyjne (pkt 9), wybór nazwy, konsultacja prawna.*

---
## Log
- **2026-07-27** — Założono projekt w vaulcie; wklejono pełny dokument koncepcyjny + analizę konkurencji (v. robocza lipiec 2026). Stan: `planning`. Następny krok: walidacja bez kodu (pkt 9).
