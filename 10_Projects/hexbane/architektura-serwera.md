---
type: project
project: Hexbane
domain: projects
status: active
created: 2026-08-31
updated: 2026-08-31
tags: [hexbane, nakama, go, architecture, match-engine]
aliases: [hexbane-server-arch]
---

# Hexbane — architektura serwera (Nakama / Go)

Stan na 2026-08-31 (`elanon1/hexbane-server@f7065fe`, working tree z usuniętym kodem OpenAI). Część projektu [[10_Projects/hexbane/_state|Hexbane]]. Liczby balansu: [[10_Projects/hexbane/zasady-gry]]. Kontrakt z klientem: [[10_Projects/hexbane/protokol-klient-serwer]].

## Szkielet

- Go 1.24, `nakama-common v1.37`, `sqlx`, `godotenv` (fatal bez `.env`), `yaml.v3`. Plugin `build/backend.so` do Nakama **3.27.0**. Brak zależności OpenAI.
- `modules/main.go` — kolejność init: `race` → `character` → `healthcheck` → `normal_match` → `ai_match` → `spellbook` → `playstyle` → `player` (legacy) → `auth` (hooki tylko logują) → `social` → `notifications` → `spell_system` → `spell_effects` → `endless_story` → rejestracja handlerów efektów.
- Konwencja modułu: `init.go` (rejestracje), `types.go`, `rpc.go`, `db.go`, `validate.go`.
- Dane: Postgres (`characters`, `character_spells`, `races`, `playstyles(+slots)`, `spells` — ta ostatnia **nie używana w runtime**); katalog zaklęć z `data/spells/<szkoła>/<tier>/<id>.yaml` (63 pliki) ładowany z `/nakama/spells` do pamięci; rasy do pamięci przy starcie.
- Migracje `db/migrations/000001..000010` (golang-migrate). **000010 czyści `characters` i `races`** i wstawia roster 35 ras. **000011_clear_races** (2026-08-31, plik do dopisania ręcznie — patrz `_state.md`) ma czyścić `characters` + `races` i zdjąć default kolumny `race_id`; fallback w kodzie to `race.DefaultRaceId` (`modules/race/types.go`, obecnie `""`).

## RPC (29 aktywnych)

Postać: `create_character` (imię 3-20, rasa, 400 pkt min 10, dokładnie 4 unikalne zaklęcia), `debug_create_character` (env `HEXBANE_ENABLE_DEBUG_RPCS`), `get_my_character`, `get_character_by_id`, `allocate_stat_points` (kontrola sumy vs unspent zakomentowana), `get_progression` · Rasy: `get_races`, `get_race` · Spellbook: `get_spellbook`, `learn_spell` (istnieje, nienauczone, wolny slot, poziom ≥ wymóg, MP ≥ koszt, transakcja), `get_available_spells`, `get_player_spells`, `get_starter_spells` · Spell system: `get_entry_spells`, `get_my_spells`, `get_spell`, `get_spell_details_yaml` · Playstyle: CRUD (≤6 slotów, kontrola właściciela) · Social: `find_friend`, `remove_friend` · Notyfikacje: `notifications_list`, `notifications_delete` · `healthcheck` (statyczne OK) · Mecze: `create_ai_arcane_duel`, `decline_match`, `create_character_match_story`.

**Zakomentowane**: `start_story`, `create/continue_character_story`, 13× `social_*` (status/follow/chat na wbudowanych API Nakamy). `RPCs.md` w root repo — nieaktualny; aktualne: `docs/API-REFERENCE-v2.md` (poza przykładami ras).

## Silnik meczu (`modules/match/`)

- `normal_match` = mecz Nakama **`"normal"`** (matchmaker: wymuszone 2/2, query `""`); `ai_match` = **`"ai_duel"`** (bot `UserId "0000"`, 150 HP/many, losowy spellbook dopasowany do liczby slotów gracza). Oba współdzielą `engine/core/loop.go` i `engine/core/player_setup.go` (`BuildPlayerState`: postać + rasa → formuły `combat`).
- **Fazy** (`engine/phase/`): `Connecting` (op0 co 1 s, op2 od obu → op1; **losowy pierwszy drafter**) → `LobbyPicking` (35 s na gracza, licznik chodzi tylko w jego turze; ściśle naprzemiennie; timeout → auto-losowanie; op70 dopiero po `CLIENT_READY{lobby_ready}`) → `LobbyCountdown` (15 s) → `Loading` (1 tick) → `GameCountdown` (2 s; op10 wysyła **oba prywatne widoki** — `todo`) → `Combat` (**300 s**) → `CombatEnd`.
- **Tick walki** (10/s): czas (op11 1/s) · koniec gdy ktoś martwy lub czas 0 · AI co 5 ticków · zakończenie castu (`EndsAt`) → `ApplySpell` → op24 · medytacja 1/s · **pasywna regeneracja many 1/s** (1.0→3.0 wg skilla) · **pasywna regeneracja HP co 3 s** · op22 co 5 ticków · sprzątanie wygasłych efektów → op15 · op14 co 500 ms lub natychmiast po usunięciu · op12 co 200 ms jeśli snapshot się zmienił · `EffectQueue.Process` · op13 log.
- **Cast**: op21 `{spell_id}` → czas = `base × (1 − CastingTimeModifier/100)` (min 0,1 s; DEX + rasa złożone przy setupie) → `TryCastSpell` atomowo (AlreadyCasting / Paralyzed / Dead / NoMana) → mana zdjęta od razu; przerywa medytację. `ApplySpell`: **refleksja 100 % na sztywno** (cel ma `reflection` ∧ zaklęcie ma efekt na wroga), zużywana (op15 `consumed`).
- **Medytacja**: op25 → warunki (żywy, nie castuje, nie sparaliżowany, nie zatruty, mana < max) → op27/26; co sekundę `+rand(10..30) × (1 + skill/200) × dexBonus`; pełna mana → op28.
- **GameOver**: HP decyduje (remis możliwy); XP, level-up, skill gains, `UpdateMatchResult`; **op50 unicast per gracz** z bogatym payloadem; boty niepersystowane; op199 → terminate.
- **AI bota** (`phase/game/ai.go`): 10-poziomowe drzewo priorytetów (heal ≤30 % → tarcza <70 % → paraliż gdy wróg castuje/mana>60 % → trucizna → refleksja → heal ≤60 % → tarcza → paraliż → najlepszy damage → cokolwiek) + reguły medytacji. Brak poziomu trudności.
- Stan: `MatchState` (RWMutex „defensywny” — jeden goroutine na mecz), `PlayerState` (z polami żywiołów rasy), `MatchStats` nigdy nie wypełniany; `TransitionTo` omijane przez większość faz; `engine/player/` legacy.

## System zaklęć (`modules/spell_system/`)

- `Spell{ID, Name, Level, Type, School, Icon, CastingTime, ManaCost, MagicPointCost, LevelRequirement, Effects[], Description, Assets*}`; `Effect{Type, Value, Duration, Delay, Interval, Target(self|enemy|ground|both)}`.
- Typy efektów: `damage, heal, poison, reflection, shield, stun, slowdown, absorb, mana_drain, paralyze, cure`. **Handlery tylko dla 7** (brak `stun`, `slowdown`, `absorb`, `mana_drain`).
- `EffectQueue` per mecz: klucz `cel → spellID/effectType`; recast tego samego zaklęcia resetuje, różne zaklęcia tego samego typu stackują; cykl `start|tick|end` („effect phase” z ostatniego commita).
- Handlery: damage (pełna formuła z `combat`, magery gain), heal (INT), shield (nadpisuje), poison (płaskie ticki, bez odporności), paralyze (łamany każdym obrażeniem), reflection (100 %), cure (zdejmuje truciznę i jej ticki).
- `docs/spell_system/GUIDE-v2.md` opisuje archiwalny prototyp 10 zaklęć (`bkp/`).

## Endless Story

Został tylko mecz `create_character` (`v2_create_character`, fazy connecting → entrymessage → createcharacter → done, opcody 100/101, jeden gracz) i typy `story_model/`. `_old_create_character/` i `ai_api/` usunięte w working tree.

## Docs w repo

`docs/DOCUMENTATION-INDEX.md` (v2, 2026-01-30) · `docs/progression/*` (autorytatywne dla balansu, obowiązek aktualizacji przy zmianach) · `docs/opcodes/*` (agent `opcode-docs` synchronizuje do klienta) · `docs/client/effect-sync.md` (op14 = pełny snapshot, op15 = podpowiedź kosmetyczna) · `docs/plans/level-up-notifications.md` (niezrobione: `Nk` w `MatchState`, kod 4) · `docs/devlog.md` (PL/EN, ostatni wpis 31.12.2025).
