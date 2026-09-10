---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, protocol, opcode, client-ready]
sources: ["client:docs/opcodes/op_02_client_ready.md", "server:docs/opcodes/op_02_client_ready.md", "server:docs/client/combat-v2.md", "client:docs/opcodes/duel-v2.md"]
---

# Opcode 2 — ClientReady

| | |
|---|---|
| Server const / client enum | `OP_CLIENT_READY` / `CLIENT_READY` |
| Direction | Client → server |
| Phases that act on it | `connecting`, `lobby_picking`, `loading`, `game_countdown` (ignored in `lobby_countdown`; rejected as "unsupported combat opcode" in `combat`; ignored in `combat_end`) |
| Client sender | `client:Application/Match/Outgoing/ClientReady/ClientReadyHandler.cs` (payload built by `ClientReadyCommand.ToPayload()`) |

## Payload (client → server)

| Field | Type | Notes |
|---|---|---|
| `combat_protocol` | int | always `2` (`client:Application/Match/Outgoing/ClientReady/ClientReadyCommand.cs:19`) |
| `user_id` | string | the sender's own user id |
| `event_name` | string | which scene is ready, see below |

`match_id` is declared on the server struct (`server:modules/match/engine/match_types/client_ready.go:7`) but the client does not send it and no phase reads it.

```json
{"combat_protocol":2,"user_id":"u1","event_name":"lobby_ready"}
```

## Event names the client sends

| `event_name` | Sent from | Server reaction |
|---|---|---|
| `match_entry_data` | `client:Application/Match/Incoming/MatchEntryData/MatchEntryDataHandler.cs:42` after [[op_00_match_entry_data]] | connecting: marks sender ready |
| `lobby_ready` | `client:Game/ScenesV3/Lobby/LobbyScreen.cs:169` | lobby_picking: unicasts [[op_70_lobby_spellbook_spells]], [[op_03_lobby_update]], [[op_05_lobby_spell_selected_update]]; marks lobby loaded |
| `game_countdown_ready` | `client:Application/Match/Incoming/GameReady/GameReadyHandler.cs:33` on [[op_07_game_ready]] | game_countdown: unicasts [[op_10_game_data]] |
| `game_hud_ready` | active `ReferenceHud.SendReadyAfterPresentation` after first rendered frame and completed scene transition | loading: acknowledges the authenticated sender’s arena; all human acknowledgements start countdown. During game_countdown it answers with opcode 10 again |

## Per-phase server behaviour

- **connecting** (`server:modules/match/engine/phase/connecting/phase.go:118-133`): payload must carry `combat_protocol == 2` (`validateCombatReady`, `:135`). Otherwise the sender receives a private, reliable opcode 30 `{"kind":"protocol_rejected","required_protocol":2}` and is never marked ready. Acting user = message sender.
- **lobby_picking** (`server:.../phase/lobby/phase.go:312-331`): only when `event_name == "lobby_ready"`. Looks up the player by the payload's `user_id` (not the sender), sends spellbook + current draft state to that user's presence, sets `LobbyLoaded[user_id] = true` (required for later spellbook refreshes).
- **game_countdown** (`server:.../phase/game_countdown/phase.go:88-106`): parses only `user_id`/`match_id` (`game_countdown/types.go`), ignores `event_name`, unicasts [[op_10_game_data]] to the presence of the payload's `user_id`. Any `CLIENT_READY` during this phase triggers it.

## Client note

`ClientReadyHandler.cs:18` returns early when the local tutorial battle is active (`TutorialSession.Battle != null`); nothing is sent to Nakama then.

## Source of truth in code
- `server:modules/match/engine/match_types/client_ready.go` — struct
- `server:modules/match/engine/phase/connecting/phase.go`, `phase/lobby/phase.go`, `phase/game_countdown/phase.go` — handling
- `client:Application/Match/Outgoing/ClientReady/ClientReadyCommand.cs` — wire payload

## Loading readiness (2026-09-10)

Only `event_name=game_hud_ready` with `combat_protocol=2` is accepted. Acting player is `MatchData.GetUserId()`, never payload `user_id`. Bots do not wait. A 30-second timeout cancels via opcode 9; no forced start. No opcode numbers or payload fields changed. Server readiness barrier deployed on 2026-09-10 as `sha-b8773ad`; the corresponding client changes still require rebuilt applications.
