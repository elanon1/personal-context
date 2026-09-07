---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, duel-v2, combat, hud]
sources: ["client:docs/opcodes/duel-v2.md", "client:docs/opcodes/duel-v2-verification.md", "client:docs/client/reference-duel/README.md"]
---

# Duel v2 on the client

Wire contract: [[combat-v2]] (opcodes 29–32) and [[opcodes]]. This note covers only what the Godot client does with it.

## Scene and versions

- Active duel scene for PvP **and** AI: `Game/ScenesV3/ReferenceDuel/MainReference.tscn` with `ReferenceHud` (`SceneManager.cs:30`). The older `Game/ScenesV3/GameHud/` HUD is kept for `Dev/GameHudPreview` / `Dev/HudPreview` only.
- Supported versions: combat protocol **2**, ruleset `duel_v2`, catalog `duel_v2.2`, 100 ms ticks (`DuelVersion.Supported`, `Core/Match/DuelV2.cs`; `DuelState.Remaining` divides ticks by 10, line 89). An unsupported snapshot throws "Unsupported combat version. Update the client." (line 98).
- Readiness (opcode 2) always sends `{combat_protocol: 2, user_id, event_name}` (`ClientReadyCommand.cs:17-21`). Event names in use: `lobby_ready` (`LobbyScreen.cs:169`, triggers the opcode 70 draft library), `game_countdown_ready` (`GameReadyHandler.cs:33`), `game_hud_ready` (`ReferenceHud.cs:197`, `GameHudScreen.cs:286`, triggers opcode 10 loadout). The lobby subscribes to events before sending readiness.

## Sending commands (opcode 29)

`DuelProtocol.Send(kind, spellId)` (`Application/Match/DuelProtocol.cs`) refuses to send unless `MatchContext.Combat.ProtocolAccepted` is true, the snapshot is not `over`, and a match id exists (line 30). `DuelState.Next` (`DuelV2.cs:95`) assigns `client_seq` from a match-scoped counter; `spell_id` is only present for `cast`. Payload: `{"client_seq":1,"kind":"cast","spell_id":"firebolt"}`; `meditate` and `clear_queue` omit `spell_id`.

## Receiving (opcodes 30–32)

`MatchMessageHandler.HandleAsync` forwards 30–32 straight to `DuelProtocol.Receive` and drops 11–15 and 21–28 (`MatchMessageHandler.cs:22-23`, `:102`); the old handler folders under `Application/Match/Incoming/Gameplay*` are dead code.

`DuelState` (`Core/Match/DuelV2.cs:80-107`):
- `Apply(snapshot)` rejects a snapshot older than the current one by `(server_tick, event_seq)` (line 99); accepting one raises the event watermark and the client sequence to `last_client_seq` (line 100). No HP/mana prediction anywhere.
- `AcceptEvent(seq)` ignores anything at or below the watermark (line 102); gaps are legal.
- `Tick` interpolates monotonic time since the last snapshot, capped at 0.5 s (line 88); `PresentationPaused` freezes it (used by the tutorial).
- `BeginMatch(id)` resets sequences only when the match id changes (line 90), so a socket replacement keeps the sequence.
- Snapshot/event DTOs are in the same file (`DuelSnapshot`, `DuelPlayer`, `DuelAction`, `DuelEvent`).

Events → visuals: `Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` (`spell_impact`, `spell_reflected`, `effect_removed` with reason `consumed|broken|depleted` shatters a `MirrorWard`). Spell id → FX id mapping is in [[spell-vfx-configuration]].

## Reconnect

`MatchManager` subscribes `GameEvents.ConnectionEstablished += RejoinCurrentMatch` (`Application/ArcaneDuel/Normal/MatchManager.cs:28, 72-79`); a failed rejoin raises `OnMatchRestoreFailed` and the HUD keeps a retry control until a fresh snapshot arrives. Character/match state survives transport loss because `MatchContext` is only reset on match end.

## HUD (`ReferenceHud.cs`, `ReferenceHud.DuelV2.cs`, `ReferenceSpellSlot.cs`)

- 6 draftable slots, 7 for Human (`StatAllocation.BaseSpellSlots = 3` plus lobby capacity from the payload; `RequiredStarterSpells` in creation is 3/4, `CreateCharacterScreen.cs:378`), plus the two standards `magic_arrow` and `mirror_reflection` (`Core/Spells/StandardSpells.cs:17-20`), deduplicated by id (`DuelLoadout.DistinctById`).
- Fillable count comes from `spell_slots`, unlocked entitlement from `max_spell_slots`; empty unlocked slots show "Unlocked · no learned spell available", locked ones "Locked by progression" (`ReferenceHud.cs:699-703`).
- Keys: `1`–`9` cast slot N (left to right, then standards), `Space` or `M` meditate, `X` sends `clear_queue` (`ReferenceHud.cs:605-607`). Swipe up also meditates. Fixed meditate and clear-queue buttons exist.
- Gating: mana, casting, paralysis, death, loading gate both buttons and shortcuts; casting/recovery do **not** block queuing. Status rows: mirror charge, barrier capacity, hex deadline, paralysis, control immunity.
- `TutorialControl(key)` / `TutorialTarget(key)` expose slot rects for the tutorial (`ReferenceHud.cs:215-221`, key `standards` merges both standard slots).
- Arena is chosen per match by `ArenaCatalog.ForMatch(matchId)` (see [[vfx-and-race-animation]]).

## Character creation and catalog

- Wizard sends `spell_ids` (`CreateCharacterCommandHandler.cs:49`): exactly 3 picks, 4 for Human, from the six `get_starter_spells` results. Standards never appear in the pick list.
- All 14 server ids are known to the client (`Core/Spells/Spell.cs:88-96` icon mapping, `SpellEffectManager.DuelV2.cs:16-20` FX mapping). Fixed 200 HP / 100 mana; stat/skill/race combat multipliers are inactive (server side, see [[combat-v2]]).
- Spellbook timing units differ per RPC (seconds vs milliseconds) and are converted at the DTO boundary (`Application/Modules/Spell/Dto/*`). (unverified: exact conversion sites)

## Validation

```sh
dotnet build hexbane.csproj
dotnet run --project Tests/DuelV2/DuelV2.csproj                       # offline model checks
NAKAMA_URL=http://127.0.0.1:7350 dotnet run --project Tests/DuelV2/DuelV2.csproj -- ai
NAKAMA_URL=http://127.0.0.1:7350 dotnet run --project Tests/DuelV2/DuelV2.csproj -- pvp
```
Live runs need the seeded `elf@test.pl` / `dark_elf@test.pl` accounts (password `123123123`, `Live.cs:75-77`) and stop after the reconnect check; final results are covered by the server tests. Earlier logs used port 57350 for a disposable stack; use whatever `NAKAMA_URL` points at.

Headless HUD smoke (macOS):
```sh
HEXBANE_IGNORE_ENV_FILE=1 HEXBANE_TEST_COMPACT=1 HEXBANE_TEST_REFERENCE=1 \
  /Applications/Godot.app/Contents/MacOS/Godot --headless --path . res://Game/ScenesV3/Dev/DuelV2Preview.tscn --quit-after 240
```
Flags read by the dev scenes: `HEXBANE_TEST_COMPACT` (1360×612), `HEXBANE_TEST_REFERENCE`, `HEXBANE_TEST_RACE=elf` (eight buttons instead of nine), `HEXBANE_TEST_DRAFT`. `Dev/StarterSelectionCheck.tscn` covers the wizard payload (3/4 picks).

Recorded results (2026-09-06, from `docs/opcodes/duel-v2-verification.md`, logs were in `/tmp/hexbane-redesign` and are not in the repo): build 0 errors / 37 warnings; offline and live ai/pvp tests PASS; headless Human 9 buttons, Elf 8 buttons, desktop 2400×1080 PASS. Not covered: physical device touch/DPI, RTT/jitter simulation, full interactive login→game flow, audio, balance. Headless startup still reports the missing `signal_lens` autoload and an unresolved theme UID; these are pre-existing.

## Source of truth in code
- `client:Core/Match/DuelV2.cs` — versions, DTOs, `DuelState` sequencing/watermark/interpolation
- `client:Application/Match/DuelProtocol.cs` — send/receive for opcodes 29–32
- `client:Application/Match/Incoming/MatchMessageHandler.cs` — routing and retired-opcode drop
- `client:Application/Match/Outgoing/ClientReady/ClientReadyCommand.cs` — readiness payload
- `client:Game/ScenesV3/ReferenceDuel/ReferenceHud*.cs`, `ReferenceSpellSlot.cs` — HUD rules and keys
- `client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` — event → visual
- `client:Tests/DuelV2/*`, `client:Game/ScenesV3/Dev/DuelV2Preview.cs` — validation harnesses
