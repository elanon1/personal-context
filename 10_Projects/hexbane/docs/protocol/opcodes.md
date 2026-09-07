---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcodes, index]
sources: ["client:docs/opcodes/README.md", "server:docs/opcodes/README.md", "server:docs/match/communication.md"]
---

# Match opcodes (index)

Numeric values are the contract and are never renumbered. Source of numbers: `server:modules/match/engine/match_types/op_codes.go`, mirrored in `client:Core/Common/Enums/Opcodes.cs`.

Three Nakama match handlers exist:

| Handler name | Registered in | Created by | Tick rate |
|---|---|---|---|
| `normal` (PvP) | `server:modules/match/normal_match/init.go:61` | matchmaker hook `MakeMatch` (`matchmaker.go:25`), 2 players forced by `BeforeMatchmakerAdd` | 10/s (100 ms) |
| `ai_duel` (bot) | `server:modules/match/ai_match/init.go:27` | RPC `create_ai_arcane_duel` (`init.go:34`) | 10/s |
| `v2_create_character` (story) | `server:modules/endless_story/create_character/init.go:25` | RPC `create_character_match_story` (creates module `create_character`, see report) | 1/s |

Duel phases (`server:modules/match/engine/phase/phase.go:15-22`): `connecting` → `lobby_picking` → `lobby_countdown` → `loading` → `game_countdown` → `combat` → `combat_end`. (`stats` is declared but never entered.)

| Constant | Value | Where |
|---|---|---|
| Draft time per player | 35 s | `server:modules/match/engine/phase/lobby/phase.go:21` |
| Lobby countdown | 15 s | `server:.../phase/lobby_countdown/phase.go:16` |
| Game countdown | 2 (broadcasts 2, 1, 0) | `server:.../phase/game_countdown/phase.go:19` |
| Combat duration | 180 s | `server:.../phase/game/phase.go:27` |
| Join grace (no one joined yet) | 30 s | `server:.../core/loop.go:24` |
| Empty match after players left | 5 s | `server:.../core/loop.go:14` |

## Index

Status `live` = sent or accepted by server code today. `retired` = tombstone: the client enum still carries the number, no server code references it, and the client dispatcher drops it (`client:Application/Match/Incoming/MatchMessageHandler.cs:23,102`).

| Op | Server const | Client enum | Direction | Phase | Purpose | Status |
|---|---|---|---|---|---|---|
| 0 | `OpMatchEntryData` | `MATCH_ENTRY_DATA` | S→C unicast, 1/s | connecting | version handshake + `me`/`enemy` views → [[op_00_match_entry_data]] | live |
| 1 | `OP_SERVER_READY` | `SERVER_READY` | S→C broadcast | connecting → lobby_picking | both players ready → [[op_01_server_ready]] | live |
| 2 | `OP_CLIENT_READY` | `CLIENT_READY` | C→S | connecting, lobby_picking, game_countdown | versioned readiness / scene ready → [[op_02_client_ready]] | live |
| 3 | `OpLobbyUpdate` | `LOBBY_UPDATE` | S→C broadcast (+unicast on lobby_ready) | lobby_picking, lobby_countdown | draft state; countdown timer → [[op_03_lobby_update]] | live |
| 4 | `OpLobbySpellSelected` | `LOBBY_SPELL_SELECTED` | C→S | lobby_picking | pick one spell → [[op_04_lobby_spell_selected]] | live |
| 5 | `OpLobbySpellSelectedUpdate` | `LOBBY_SPELLS_SELECTED_UPDATE` | S→C broadcast | lobby_picking | picks with name/icon/description → [[op_05_lobby_spell_selected_update]] | live |
| 6 | `OpLaunchGame` | `LOBBY_START_GAME` | S→C broadcast | lobby_countdown → loading | empty cue → [[op_06_launch_game]] | live |
| 7 | `OpGameReady` | `GAME_READY` | S→C broadcast | loading → game_countdown | empty cue → [[op_07_game_ready]] | live |
| 8 | `OP_MATCH_DECLINED` | `MATCH_DECLINED` | S→C broadcast | any (`normal` only) | a player declined via RPC `decline_match` → [[op_08_match_declined]] | live |
| 9 | `OP_MATCH_CANCELED` | `MATCH_CANCELED` | S→C | any join / pre-combat leave | `{reason}` → [[op_09_match_canceled]] | live |
| 10 | `OpGameData` | `GAME_ENTRY_DATA` | S→C unicast | game_countdown | combat loadouts `me`/`enemy` → [[op_10_game_data]] | live |
| 16 | `OpGameCountdown` | `GAME_COUNTDOWN` | S→C broadcast, 1/s | game_countdown | `time_remaining` 2,1,0 → [[op_16_game_countdown]] | live |
| 29 | `OpCombatCommand` | `COMBAT_COMMAND` | C→S | combat | `cast` / `meditate` / `clear_queue` → [[op_29_combat_command]] | live |
| 30 | `OpCombatResult` | `COMBAT_RESULT` | S→C unicast, reliable | connecting (`protocol_rejected`), combat | private result for the sender → [[op_30_combat_result]] | live |
| 31 | `OpCombatEvent` | `COMBAT_EVENT` | S→C broadcast, reliable | combat | public combat events → [[op_31_combat_event]] | live |
| 32 | `OpCombatSnapshot` | `COMBAT_SNAPSHOT` | S→C unicast, reliable | combat (every 2 ticks), rejoin, combat end | authoritative state → [[op_32_combat_snapshot]] | live |
| 50 | `OpGameOver` | `GAMEOVER` | S→C unicast | combat_end | personalized result + progression → [[op_50_game_over]] | live |
| 70 | `OpLobbySpellbookSpells` | `LOBBY_SPELLBOK_SPELLS` (sic) | S→C unicast | lobby_picking | draftable spellbook → [[op_70_lobby_spellbook_spells]] | live |
| 100 | `OpStoryUpdate` | `ENDLESS_STORY_UPDATE` | S→C unicast | story match | narration + question → [[op_100_story_update]] | live, separate feature (WIP) |
| 101 | `OpStoryChoiceSelected` | `ENDLESS_STORY_CHOICE` | C→S | story match | `choice_id`; server only logs → [[op_101_story_choice_selected]] | live, WIP |
| 199 | `OpQuitGame` | `QuitGame` | C→S | combat_end | terminate match (already terminating, see doc) → [[op_199_quit_game]] | live |
| 11 | – | `GAME_TIME_UPDATE` | – | – | old game clock | retired |
| 12 | – | `GAMEPLAY_UPDATE_PLAYERS_UPDATE` | – | – | old player sync | retired |
| 13 | – | `GAMEPLAY_LOG` | – | – | old combat log | retired |
| 14 | – | `GAMEPLAY_EFFECTS_UPDATE` | – | – | old effect sync | retired |
| 15 | – | `GAMEPLAY_EFFECT_REMOVED` | – | – | old effect removal | retired |
| 21 | – | `GAMEPLAY_SPELL_CASTED` | – | – | old cast request/echo | retired |
| 22 | – | `GAMEPLAY_SPELL_RELEASE` | – | – | old "can cast" ping | retired |
| 23 | – | `GAMEPLAY_SPELL_FAILED` | – | – | old cast rejection | retired |
| 24 | – | `GAMEPLAY_SPELL_ACCEPTED` | – | – | old cast resolution | retired |
| 25 | – | `GAMEPLAY_MEDITATE` | – | – | old meditate request | retired |
| 26 | – | `GAMEPLAY_MEDITATE_FAILED` | – | – | old meditate rejection | retired |
| 27 | – | `GAMEPLAY_MEDITATE_ACCEPTED` | – | – | old meditate ack | retired |
| 28 | – | `GAMEPLAY_MEDITATE_INTERRUPTED` | – | – | old meditate interrupt | retired |

Notes:

- **Opcode 9 is not shared.** Both old doc trees said 9 was reused for the game countdown. Code says `OpGameCountdown = 16` (`server:.../op_codes.go:24`, `server:.../phase/game_countdown/phase.go:17`, `client:Core/Common/Enums/Opcodes.cs:24`). 9 is only `OP_MATCH_CANCELED`.
- **Retired opcodes 11–15 / 21–28** have no server constant at all; all old combat payload docs (`SpellCastingStatus`, `SpellStatus`, `SpellReason`, `OpGamePlayLog` types) describe code that no longer exists. The client's `SpellStatus`/`SpellCastingReason` enums in `client:Core/Spells/Spell.cs:113-129` survive only as dead types.
- The client `Opcodes` enum keeps the retired numbers and the `Application/Match/Incoming/Gameplay*` handlers exist but are never registered (dispatcher skips them).

## Server dispatch rules (`server:modules/match/engine/core/loop.go`)

- Sender must be a participant (`Players[sender]`), else dropped with a warning (`loop.go:114`).
- A message whose session id differs from the sender's current presence is dropped silently (`loop.go:119`). This is what makes a replacement socket win over the old one.
- Phases must take the acting user from `msg.GetUserId()`. (Lobby and game-countdown `CLIENT_READY` handling still read `user_id` from the payload; see report.)
- In `combat` the queued messages are dispatched **before** the phase tick; in every other phase **after** it (`loop.go:54-71`).
- Bots (`ai_duel`, user id `"0000"`, `server:modules/match/ai_match/bot.go:16`) have no presence: unicast messages to them are skipped.

## Client dispatch (`client:Application/Match/Incoming/MatchMessageHandler.cs`)

- Opcodes 30–32 → `DuelProtocol.Receive` (`client:Application/Match/DuelProtocol.cs:38`).
- 11–15 and 21–28 → dropped before lookup (`MatchMessageHandler.cs:23`) and never registered (`:102`).
- Everything else → reflection registry of `IMatchDataHandler<T>` keyed by `[MatchOpcode]` on the message DTO; empty payload instantiates the DTO with defaults.
- Outgoing senders: 2 `ClientReadyHandler`, 4 `LobbySpellSelectionHandler`, 29 `DuelProtocol.Send`, 101 `ChoiceSelectedCommandHandler`, 199 `QuitCommandHandler` (all under `client:Application/Match/Outgoing/`).

See [[combat-v2]] for the combat contract, [[shared-types]] for payload structs, [[duel-v2-client]] for the client implementation.

## Source of truth in code
- `server:modules/match/engine/match_types/op_codes.go` — every duel opcode number
- `server:modules/endless_story/create_character/types.go:31-32` — story opcodes 100/101
- `server:modules/match/engine/core/loop.go` — dispatch and termination rules
- `server:modules/match/engine/phase/*/phase.go` — which phase sends/accepts what
- `client:Core/Common/Enums/Opcodes.cs` — client enum (includes retired tombstones)
- `client:Application/Match/Incoming/MatchMessageHandler.cs` — client routing
