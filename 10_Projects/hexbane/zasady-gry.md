---
type: project
project: Hexbane
domain: [projects, creative]
status: active
created: 2026-08-31
updated: 2026-08-31
tags: [hexbane, game-design, balance, progression, spells]
aliases: [hexbane-rules, hexbane-balance]
---

# Hexbane — zasady gry i liczby balansu

Stan kodu serwera na 2026-08-31 (autorytatywne: `hexbane-server/docs/progression/*`, `modules/{progression,combat,skills}`, `data/spells/`). Część [[10_Projects/hexbane/_state|Hexbane]].

## Postać

- **Tworzenie**: 400 punktów na STR/INT/DEX, min 10 każdy (domyślnie 133/134/133) + modyfikatory rasy; dokładnie 4 zaklęcia startowe; 1 postać na konto. `MaxHealth = STR`, `MaxMana = INT`.
- **Rasy**: `Race{str/int/dex_modifier, casting_time_modifier, spell_resistances{spellId: %}, primary/secondary_element + bonus %}`. **Roster pusty od 2026-08-31** (stare 35 ras wycofane, nowe od zera); domyślna rasa `race.DefaultRaceId` = `""` do czasu nowej migracji — bez rasy postać dostaje czyste bazowe staty.
- **DEX**: skrócenie castu 2,5 %/100 DEX (+ |mod rasy|, cap 15 %), unik 1,25 %/100 (cap 5 %), regen many +0,05 %/pkt.
- **Skille** (0–100): Meditation, Spell Resistance, Magery. Szansa na +0,1: 15 % (<50), 8 % (50–75), 3 % (75–90), 0,5 % (90+). Efekt = skill/200 (max 50 %). Magery rośnie przy zadawaniu obrażeń, Resistance przy otrzymywaniu, Meditation przy ticku medytacji. Zapis do bazy po meczu.
- **XP**: wygrana 100, przegrana 30, +50 pierwsza wygrana dnia, +50 dzienny bonus (raz/dzień). `XP(level) = 100 × 1,5^(level−2)`, max 30 (poziom 30 ≈ 8,3 M XP). Za poziom: 5 punktów statów, magic points 2/4/10/15 (poziomy 1–5/6–10/11–20/21–30), sloty zaklęć: start 4, +1 na 4/6/10/15/20/25 (max 10).
- **Nauka zaklęć**: koszt w MP (domyślnie 5; tiery 1–2 = 0), wymóg poziomu, wolny slot.

## Walka

- Czas meczu **300 s**, 10 ticków/s, bez ruchu. Koniec: HP ≤ 0 albo czas (wtedy porównanie HP, remis możliwy).
- **Cast**: `czas = base × (1 − modyfikator/100)` (min 0,1 s); mana zdejmowana od razu; odrzucenie: brak many / już castuje / paraliż / martwy. Cast przerywa medytację; obrażenia **nie** przerywają castu.
- **Medytacja**: warunki — żywy, nie castuje, nie sparaliżowany, **nie zatruty**, mana < max. Co sekundę `+rand(10..30) × (1 + Meditation/200) × dexBonus`; kończy się przy pełnej manie lub caście.
- **Regeneracja pasywna** (nieopisana w docs): mana 1,0→3,0/s (wg Meditation), HP co 3 s.
- **Obrażenia**: `dmg = base + base×Magery/200 + INT×0,1`; `× (1 + bonus żywiołu/100)` jeśli szkoła zaklęcia = żywioł rasy castera; `odporność = min(75 %, Resistance/200 + statResist + odporność rasy na dane zaklęcie)`; `final = max(1, dmg × (1 − odporność))`. `statResist`: kinetic ← STR×0,0005, mind ← INT×0,0005 — ale typ obrażeń jest **na sztywno `mind`**. Trucizna tickuje płasko (bez odporności).
- **Leczenie**: `base + INT×0,05`. **Tarcza** nadpisuje wartość. **Paraliż** łamie każde obrażenie. **Refleksja**: 100 %, jednorazowa, odbija tylko zaklęcia z efektem na wroga.
- **Efekty**: kolejka per mecz z kluczem `cel → zaklęcie/typ` — recast tego samego zaklęcia resetuje czas; różne zaklęcia tego samego typu stackują się.

## Lobby (draft)

Losowy pierwszy wybierający; naprzemiennie; **35 s** na gracza (licznik chodzi tylko w jego turze); po czasie auto-losowanie; potem odliczanie 15 s; liczba slotów = sloty postaci (4 na start). Klient dodatkowo pozwala **ułożyć** wybrane zaklęcia w lewą/prawą rękę (`SpellSlotOrder`).

## Katalog zaklęć (63 = 9 szkół × 7 tierów)

Szkoły: Air, Fire, Water, Ice, Lightning, Mind, Toxic, Earth, Neutral. Tier → koszt: t1–2 0,6 s / 8 many / poziom 1 / 0 MP; t3–4 1,0 s / 14 / 3; t5 1,4 s / 22 / 5; t6–7 1,8 s / 30 / 8.

Typy efektów: `damage, heal, poison, reflection, shield, stun, slowdown, absorb, mana_drain, paralyze, cure`. **Zaimplementowane handlery: 7** — `stun`, `slowdown`, `absorb`, `mana_drain` nie działają (błąd w logu), co dotyka ~18 zaklęć: repulse, zephyr_veil, air_shift, skysunder, crash, whiteout, deep_freeze, glacial_barrier, static_charge, essence_drain, enfeeble, confusion, terror, nerve_venom, boulder_strike, grasping_roots, quake, blessing.

Ciekawostki balansu: `mirror_ward` (Neutral t1, 8 many) = refleksja **120 s**; `stormguard` (Lightning t7) = refleksja 10 s; największe uderzenia `discharge` 54, `lance_of_winter` 45, `venom_rupture` 45; `paralyze` (Mind t1) = 10 s paraliżu za 8 many.

Opisy zaklęć są po polsku (`description`), `data/spells - spells.csv` to arkusz projektowy (nie czytany przez kod).

## Bot

Drzewo priorytetów: heal ≤30 % HP → tarcza gdy <70 % → paraliż gdy wróg castuje lub ma >60 % many → trucizna → refleksja → heal ≤60 % → tarcza → paraliż → najsilniejszy damage → cokolwiek; medytuje gdy mana <10, wróg sparaliżowany, wróg w caście >0,8 s lub nie stać go na najtańsze zaklęcie. Bot: 150 HP / 150 many, losowy spellbook. Brak poziomów trudności.
