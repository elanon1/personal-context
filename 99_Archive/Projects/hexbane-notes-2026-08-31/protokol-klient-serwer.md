---
type: project
project: Hexbane
domain: projects
status: archived
created: 2026-08-31
updated: 2026-09-07
archived: 2026-09-07
original-path: 10_Projects/hexbane/protokol-klient-serwer.md
superseded-by: "[[_index]] (10_Projects/hexbane/docs/)"
tags: [hexbane, nakama, protocol, opcodes, rpc]
aliases: [hexbane-opcodes, hexbane-rpc]
---

> [!warning] Zarchiwizowane 2026-09-07 — opis sprzed przeprojektowania duel_v2. Aktualna dokumentacja: `10_Projects/hexbane/docs/` (patrz `[[_index]]`). Audyt rozbieżności: `[[2026-09-07-vault-notes-audit]]`.

# Hexbane — protokół klient ↔ serwer

Stan na 2026-08-31, zweryfikowany po obu stronach ([[10_Projects/hexbane/architektura-klienta]] · [[10_Projects/hexbane/architektura-serwera]]). Źródła prawdy: klient `Core/Common/Enums/Opcodes.cs`, serwer `modules/match/engine/match_types/op_codes.go`, docs `docs/opcodes/*.md` (obie strony, synchronizowane z serwera).

## Opcody (numeracja identyczna po obu stronach)

| Op | Nazwa | Kierunek | Faza | Payload / uwagi |
|---|---|---|---|---|
| 0 | MatchEntryData | S→C | Connecting | `{me: PrivatePlayerView, enemy: PublicPlayerView}` co 1 s |
| 1 | ServerReady | S→C | Connecting | oba `CLIENT_READY` odebrane |
| 2 | ClientReady | C→S | wiele | `{user_id, event_name}`: `match_entry_data`, `lobby_ready` (odblokowuje op70), `game_hud_ready` (odblokowuje op10) |
| 3 | LobbyUpdate | S→C | Lobby | `{time_remaining, player_timers, draft_turn_user_id, spells_selected, spell_slots, slots_left}`; też odliczanie LobbyCountdown |
| 4 | LobbySpellSelected | C→S | Lobby | `{spell_id}` |
| 5 | LobbySpellSelectedUpdate | S→C | Lobby | `{spells_selected: {userId: [Spell]}, spell_slots}` |
| 6 | LaunchGame | S→C | LobbyCountdown→Loading | |
| 7 | GameReady | S→C | Loading | |
| 8 | MatchDeclined | S→C | pre-combat | sygnał `decline` |
| 9 | MatchCanceled | S→C | pre-combat | `{reason}` — gracz wyszedł |
| 10 | GameData | S→C | GameCountdown | `{me, enemy}` — **oba prywatne widoki** |
| 11 | GameTimeChange | S→C | Combat | `{time_remaining}` co 1 s |
| 12 | GameUpdatePlayers | S→C | Combat | `{userId: PlayerSnapshot}` co 200 ms przy zmianie |
| 13 | GamePlayLog | S→C | Combat | `{type, amount, who[]}` |
| 14 | GameEffectsUpdate | S→C | Combat | `{userId: [{key, effect, icon, remove_after}]}` co 500 ms / natychmiast po usunięciu; **pełny snapshot, zastąp** |
| 15 | GameEffectRemoved | S→C | Combat | `{user_id, effect, reason}`; reason `expired|broken|depleted|consumed|cured`; tylko do FX |
| **16** | GameCountdown | S→C | GameCountdown | `{time_remaining}` — **docs mówią 9, kod 16** |
| 21 | SpellCasted | oba | Combat | C→S `{spell_id}`; S→C status `Casting` `{spell, status, reason, identifier, sender}` |
| 22 | SpellCastingRelease | S→C | Combat | co 5 ticków |
| 23 | SpellFailed | S→C | Combat | `reason`: None 0, NoMana 1, AlreadyCasting 2, Interrupted 3, Paralyzed 4, Dead 5, NotHandled 6 |
| 24 | SpellAccepted | S→C | Combat | + `{was_reflected, reflection_percent, final_target}` |
| 25 | Meditated | C→S | Combat | |
| 26/27/28 | MeditationFailed/Accepted/Interrupted | S→C | Combat | klient: puste handlery |
| 50 | GameOver | S→C | CombatEnd | unicast per gracz `{outcome, reason, opponent, xp{}, level_up{}, skill_gains{}, stats{}, record{}}` |
| 70 | LobbySpellbookSpells | S→C | Lobby | `{availableSpells: [{spell, used}]}` |
| 100 / 101 | StoryUpdate / StoryChoiceSelected | S→C / C→S | Endless Story | `{narration, question}` / `{choice_id}` |
| 199 | QuitGame | C→S | Combat | terminate |

`PlayerSnapshot`: `user_id, mana, mana_max, hp, hp_max, shield, casting, meditating, paralyzed, poisoned, effects[]`. `SpellStatus`: Rejected 0, Accepted 1, Casting 2.

## RPC — mapowanie klient → serwer

| Klient woła | Serwer | Status |
|---|---|---|
| `create_character`, `allocate_stat_points`, `get_progression`, `get_my_character`, `get_character_by_id` | moduł `character` | OK |
| `get_races` | `race` | OK |
| `get_player_spells`, `get_spellbook`, `get_starter_spells`, `learn_spell` | `spellbook` | OK |
| `get_spell` | `spell_system` | OK |
| `find_friend`, `remove_friend` | `social` | OK |
| `notifications_list`, `notifications_delete` | `notifications` | OK |
| socket: `create_ai_arcane_duel`, `decline_match`, `create_character_match_story` | `ai_match`, `normal_match`, `endless_story` | OK |
| `start_story` (`StartStoryQuery`) | **zakomentowane** | ❌ |
| `get_users` (`GetUsersQuery`) | **nie istnieje** | ❌ |

Serwer ma też RPC, których klient nie woła: `get_entry_spells`, `get_my_spells`, `get_available_spells`, `get_spell_details_yaml` (używane przez n8n/`vfx.md`), `get_race`, playstyle CRUD, `healthcheck`, `debug_create_character`.

## Nazwy meczów Nakama

`"normal"` (PvP, matchmaker 2/2, query `""`) i `"ai_duel"` (bot) — nie `ad_normal`/`ad_ai` (to klucze DI po stronie klienta). Mecz Endless Story: `create_character`.

## Zasady synchronizacji efektów (z `docs/client/effect-sync.md`)

op14 = autorytatywna pełna lista per gracz (pusta tablica ≠ brak zmiany — wyczyść); odliczanie z `remove_after` względem czasu serwera (op11), nie z lokalnego timera; op15 tylko do odpalenia FX; po reconnect zakładaj zero efektów do następnego snapshotu. Klient (`GameHudScreen.RebuildEffects`, `EffectSlot`) realizuje to zgodnie z dokumentem.
