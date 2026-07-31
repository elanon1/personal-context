---
type: project
domain: [personal-life]
status: active
created: 2026-07-31
updated: 2026-07-31
tags: [english, learning, career]
aliases: [english-c1, angielski]
project: english-c1
state: active
---

# English C1 — program 6 miesięcy

> Cel: **bliski C1 w mówionym angielskim do końca stycznia 2027**, pod dwa konkretne zastosowania: **rozmowy rekrutacyjne** (screeny, behavioralne STAR, techniczne deep-dive, negocjacje stawki) i **small talk** (openery, networking, naturalne reakcje). Wspiera cel z [[goals]]: dołączenie do silnej inżyniersko organizacji.

## Stan wyjściowy (learner model, 2026-07-31)

Źródło prawdy: Postgres `agent_db.english_profile` — oceny na podstawie realnych wypowiedzi, aktualizowane automatycznie przez coacha.

| Wymiar | Wynik | CEFR | Obserwacje |
|---|---|---|---|
| fluency | 48/100 | B2 | pod presją jedno długie run-on zdanie zamiast segmentacji na kroki |
| accuracy | 58/100 | B2+ | wzorzec "by + -ing" łamie się pod presją; mylone rollback (rzecz./czas.) |
| range | 56/100 | B2+ | dobry techniczny słownik, miejscami naturalne idiomy; kalki ze składni polskiej w dłuższych frazach |
| listening | 50/100 | ? | jeszcze nieocenione |
| presentation_qa | 50/100 | ? | jeszcze nieocenione |
| interview | — | ? | nowy wymiar, w kalibracji |
| smalltalk | — | ? | nowy wymiar, w kalibracji |

SRS: 36 fraz w obiegu (boxy Leitnera 1/3/7/21/60 dni). Log praktyki: 32 interakcje od 2026-07-28.

## System nauki (w pełni zautomatyzowany, n8n + Telegram)

Dedykowany bot **English Teacher** — 100% wiadomości to trening, z pamięcią rozmowy (wieloturowe mock interviews i roleplay), głos (STT) i tekst. Learner model + SRS + log w Postgres `agent_db`.

- **07:30 codziennie** (audio): pn/śr/pt pytanie rekrutacyjne pod aktualną fazę; wt/czw scenka small talk; weekend swobodna płynność + powtórki.
- **~3× dziennie** dropy: prosta fraza → ostrzejsza natywna (rotacja: interview / small talk / meetingi) → SRS.
- **20:30 codziennie** Vocab Builder: aktywna odpytka z zaległych SRS (bez podpowiedzi) + 2-3 nowe frazy pod fazę. ~150 fraz/mies.
- **Piątek 16:00** test tygodnia z realnego materiału + finałowe pytanie rekrutacyjne; **pierwszy piątek miesiąca** = checkpoint: aktualizacja profilu + szczera ocena dystansu do C1.

## Roadmapa

- **Faza 1 — sierpień 2026:** kalibracja + dopracowane 60-90s "tell me about yourself", openery small talku, nawyk mówienia codziennie.
- **Faza 2 — wrzesień-październik:** behawioralne STAR, narracja projektów/incydentów, opinie i trade-offy, zasięg idiomatyczny.
- **Faza 3 — listopad-grudzień:** presja: pełne mock interviews, trudne pytania, negocjacje, uprzejma niezgoda, szybkie natywne słuchanie, struktury C1.
- **Faza 4 — styczeń 2027:** konsolidacja i weryfikacja — cotygodniowy pełny mock interview, domknięcie słabych wymiarów.

## Log

- **2026-07-28** — start systemu: coach (sub-agent w Personal Assistant), learner model w Postgres, Daily/Drops/Weekly.
- **2026-07-31** — przeorientowanie na C1/rekrutację/small talk (fazy, nowe tryby interview+smalltalk, 7 wymiarów profilu); doszedł wieczorny Vocab Builder; angielski wydzielony do dedykowanego bota English Teacher z pamięcią rozmowy; wszystkie wysyłki przepięte na nowego bota.
