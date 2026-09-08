---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
tags: [hexbane, client, duel-v2, combat, hud]
sources: ["client:docs/opcodes/duel-v2.md", "client:docs/opcodes/duel-v2-verification.md", "client:docs/client/reference-duel/README.md"]
---

# Duel v2 on the client

Wire contract: [[combat-v2]] (opcodes 29–32) and [[opcodes]]. This note covers only what the Godot client does with it.

## Catalog2.4 and progression client preparation (2026-09-08)

The compatible implementation is applied to the actual client repository `/Users/elanon/RiderProjects/hexbane`. Its `dotnet build --no-restore` passes (0 errors, 9 existing warnings), and `Tests/Progression` passes shared-budget and combat-preview tests. No live deployment or interactive Godot/network verification of these new controls has occurred.

`DuelVersion.Supported` accepts catalog `duel_v2.4` and retains 2.2/2.3 for existing tutorial/preview fixtures, still protocol 2/rulesetduel_v2. Server spell views provide selected primary effects and effective cost/cast/recovery; snapshots provide actual maxima. Mirror live `remaining` is one charge; definition `value` is return percentage. A reflected packet can do zero damage after rounding; Arrow remains fixed 1. Formulas and reflection are in [[combat-stat-rules]]. Local tutorial remains a separate training model.

Character Spellbook gains **PRIMARY DEVELOPMENT**: a scrollable six-tier path selector uses server node IDs and `next` edges, shows locked tiers/checkpoints, clears later choices when an earlier choice changes, and saves via `set_primary_path`. Saved effective base config displays mana/cast/recovery plus mirror window/return. Both primary paths are independent. Reopening fetches current config; backend failures are displayed.

Stats gains **REALLOCATE ALL STATS**: three numeric inputs spend exactly400+5×(level−1), with min 10 each. Saving calls `respec_stats`, clears local pending allocation and reloads details. Creation and incremental allocation remove racial ceilings/flat grants; fallback race traits and previews use shared softened stats and migration 000005 values. Active-match build changes fail server-side and show their error; no optimistic mutation is committed.

The character progression view consumes `study_xp`, `ranked_eligible`, `primary_tier`. At cap the XP bar becomes **Spell study /500**, explaining +5MP and ranked access. Dashboard mode selection adds ranked, disabled below character level 30; skills, MP, collection and primary allocation are not consulted. Normal MatchManager carries `queue=normal|ranked` plus matching query, default normal; the server remains authoritative. This is queue/access support, not rating/MMR UI.

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

`DuelProtocol.PresentationSpell` reads server spell data first and falls back to `SpellPresentationCatalog.Find` for missing metadata (standards and Firebolt); arena previews use the same resolver for cast timing/visual keys.

Events → visuals: `Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` handles MirrorWard removal/reflection shatter; `SpellEffectManager.Projectiles.cs` handles Magic Arrow and Firebolt release, reflection, impact/dodge and snapshot reconciliation. Their short flights are cosmetic because both server spells have zero travel time. Spell id → FX id mapping is in [[spell-vfx-configuration]].

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
- All 14 server ids are known to the client (`Core/Spells/Spell.cs:88-96` icon mapping, `SpellEffectManager.DuelV2.cs:16-20` FX mapping). Server resources and stat/skill/trait modifiers are active and authoritative; the former fixed 200/100 prototype is superseded (see [[combat-v2]]).
- Spellbook timing units differ per RPC (seconds vs milliseconds) and are converted at the DTO boundary (`Application/Modules/Spell/Dto/*`). (unverified: exact conversion sites)

## Prepared implementation validation (2026-09-08)

`dotnet build hexbane.csproj --no-restore` passes in the temporary copy with 0 errors and 9 existing warnings. `dotnet run --project Tests/Progression/Progression.csproj --no-restore` passes shared-budget, universal-floor, legacy-race-bounds and softened-preview checks. `git apply --check` passed against the actual client. These are compile/pure-behavior checks, not confirmation of live RPC success, visual fit or deployed migration state.

## Existing validation harnesses

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

Historical results (2026-09-06, from `docs/opcodes/duel-v2-verification.md`, logs were in `/tmp/hexbane-redesign` and are not in the repo): build 0 errors / 37 warnings; offline and live ai/pvp tests PASS; headless Human 9 buttons, Elf 8 buttons, desktop 2400×1080 PASS. Not covered: physical device touch/DPI, RTT/jitter simulation, full interactive login→game flow, audio, balance. Headless startup still reports the missing `signal_lens` autoload and an unresolved theme UID; these are pre-existing.

## Source of truth in code
- Prepared client: `Application/Modules/Character/PrimaryProgression/PrimaryProgression.cs`, `Game/ScenesV3/CharacterDetail/CharacterDetailScreen.Progression.cs` — graph/respec RPCs and UI
- Prepared client: `Game/ScenesV3/Dashboard/ModeOverlay.cs`, `Application/ArcaneDuel/Normal/MatchManager.cs` — ranked mode and queue
- Prepared client: `Core/Characters/{StatAllocation,RaceCatalog,RaceTraits}.cs`, `Tests/Progression` — shared rules/preview validation
- `client:Core/Match/DuelV2.cs` — versions, DTOs, `DuelState` sequencing/watermark/interpolation
- `client:Application/Match/DuelProtocol.cs` — send/receive for opcodes 29–32
- `client:Application/Match/Incoming/MatchMessageHandler.cs` — routing and retired-opcode drop
- `client:Application/Match/Outgoing/ClientReady/ClientReadyCommand.cs` — readiness payload
- `client:Game/ScenesV3/ReferenceDuel/ReferenceHud*.cs`, `ReferenceSpellSlot.cs` — HUD rules and keys
- `client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` — event → visual
- `client:Tests/DuelV2/*`, `client:Game/ScenesV3/Dev/DuelV2Preview.cs` — validation harnesses
