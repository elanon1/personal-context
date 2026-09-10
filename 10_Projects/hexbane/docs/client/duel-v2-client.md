---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, client, duel-v2, combat, hud]
sources: ["client:docs/opcodes/duel-v2.md", "client:docs/opcodes/duel-v2-verification.md", "client:docs/client/reference-duel/README.md"]
---

# Duel v2 on the client

Wire contract: [[combat-v2]] (opcodes 29–32) and [[opcodes]]. This note covers only what the Godot client does with it.

## Catalog2.4 progression UI (2026-09-08, Claude Code)

Implemented and verified live against local Nakama (plugin with `get_primary_progression`/`set_primary_path`/`respec_stats`, migrations 000004/000005 applied) on the 1360×612 phone viewport and 1920×1080 desktop. Round trips persisted: a saved mirror path landed in `characters.primary_paths`, a respec landed in `strength/intelligence/dexterity` with `unspent_stat_points=0`. Rendering/verification path: `Game/ScenesV3/Dev/ProgressionVerification.tscn` (see harness list below).

`DuelVersion.Supported` accepts catalog `duel_v2.4` and retains 2.2/2.3 for existing tutorial/preview fixtures, still protocol 2/rulesetduel_v2. Server spell views provide selected primary effects and effective cost/cast/recovery; snapshots provide actual maxima. Mirror live `remaining` is one charge; definition `value` is return percentage. A reflected packet can do zero damage after rounding; Arrow remains fixed 1. Formulas and reflection are in [[combat-stat-rules]]. Local tutorial remains a separate training model.

**Character screen — fourth tab `Primary`** (`CharacterDetailScreen.Primary.cs`, scene `PrimaryView`):
- *Progression ladder* (`ProgressionCurve.Milestones`): one chip per unlocking level — L1/5/10/16/23/30 primary tiers (3 and 6 marked as checkpoints), L7/11/16 draft slots (Human counts +1), L30 ranked. Reached chips lit, the next one framed in accent, later ones dimmed. One-line status: XP to next level, next tier/MP/slot level, ranked level, 120/70 XP per result; at cap: study XP /500 → +5MP and "Ranked unlocked".
- *Two path editors* (Magic Arrow, Mirror Reflection) built from the server graph (`nodes[].id/tier/next/modifier`): six tier rows with `T<n> · LV<level>` badges, 1–2 tappable node chips per row (selected = accent frame + check, locked = dimmed + lock, checkpoint = gold hint). Tapping a tier truncates later choices. The header shows *Now:* (or *Pending:* in accent with *Saved:* under it) folded client-side by `Core/Spells/PrimaryPath.Fold` from the catalog baseline + node modifiers; the server `config` is used for the saved line when present. `SAVE PATH` (PlaqueButton) / `REVERT`; status explains tiers still waiting. The answer to `set_primary_path` replaces local state, then details are reloaded.
- Tab badge "N new" and the dashboard panel **Primary Path Ready** (`DashboardScreen.RefreshPrimaryReady`) count primaries whose saved path is shorter than the earned tier; both route into the Primary tab (`CharacterDetailTab.Primary`).
- Spellbook detail button for a standard spell reads `Primary · tier N/6 › Develop` and switches to the tab.

**Stats tab — inline reallocation** (`CharacterDetailScreen.Respec.cs`): `REALLOCATE ALL POINTS` (GhostButton under the rows, hidden while a level-up allocation is pending) flips the same three rows into full-budget mode (400+5×(level−1), unspent points join the pool, min 10 per stat). +/− repeat while held (`AddHoldRepeat`, 0.4 s then accelerating); the pill shows `remaining / budget`; the confirm button reads `Place N more` until the pool is empty, then `Save new build` → `respec_stats`; `CANCEL` restores. The preview card diffs saved → proposed. Values above 200 turn gold with a tooltip (soft cap). The old popup `Window` implementation (`CharacterDetailScreen.Progression.cs`) was removed.

**Visibility elsewhere:** XP header at level 30 shows `Spell study N / 500 XP → +5 MP · Ranked unlocked`; character meta line ends with `RANKED` or `Ranked at LVL 30`; the summary race line adds the race trait sentence; server modifiers render with unit suffixes (`%`, ` mana/s`, `×`, ` slots`) and zero rows are hidden. Spellbook subtitle: `N MP available · owned/optional spells · draft slots N, next at level L · next +MP at level L`. Dashboard subtitle: `Level N · XP to next` or `Level 30 · Ranked`; 3-digit badges shrink. Mode overlay: `vs Player · Ranked · vs AI` in the scene; Ranked is disabled with a lock glyph and a hint line (`unlocks at level 30 — you are level N, X XP to go`) — level is the only gate. Game over: level-up line lists `+MP`, `draft slot N` only when 7/11/16 was crossed, `primary tier N` when a tier level was crossed, `RANKED UNLOCKED` at 30; at cap the next-level line says results feed spell study.

`ProgressionCurve` (Core) mirrors server constants (T(L)=45(L−1)+6(L−1)(L−2), MP 5/5/5/2 schedule, slots 7/11/16, tiers 1/5/10/16/23/30, budget 400+5/level, soft cap 200) for hints and the ladder; server numbers win whenever `get_character_details` answers. Unit checks live in `Tests/Progression`.

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
- Keys are configurable and persisted locally: draft 1–5/R/F, Magic Arrow Q, Mirror Reflection E, meditate Space, clear queue X by default. Primary bindings are stable across race capacities. Auto/Desktop/Mobile HUD profiles and in-duel Esc settings are described in [[combat-ui-profiles]]. Swipe up also meditates; both profiles retain clickable actions.
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
- `client:Application/Modules/Character/PrimaryProgression/PrimaryProgression.cs` — graph/respec RPC DTOs (modifier, full config)
- `client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.{Primary,Respec}.cs` + `CharacterDetailScreen.tscn` (`PrimaryTab`, `PrimaryView`, `ReallocateBtn`) — Primary tab, ladder, path editors, inline respec
- `client:Game/ScenesV3/Dashboard/{ModeOverlay.tscn,ModeOverlay.cs,DashboardScreen.cs,DashboardScreen.tscn}` — ranked gate, Primary Path Ready panel
- `client:Core/Characters/ProgressionCurve.cs`, `client:Core/Spells/PrimaryPath.cs`, `client:Tests/Progression` — curve constants, node text/fold, unit checks
- `client:Game/ScenesV3/Dev/ProgressionVerification.{cs,tscn}` — live walkthrough + PNG captures (`AUTH_EMAIL/AUTH_PASSWORD`, `PROGRESSION_SIZE=phone|pc`, `PROGRESSION_CAPTURE=<prefix>`, `PROGRESSION_SAVE_RESPEC=1`)
- `client:Core/Match/DuelV2.cs` — versions, DTOs, `DuelState` sequencing/watermark/interpolation
- `client:Application/Match/DuelProtocol.cs` — send/receive for opcodes 29–32
- `client:Application/Match/Incoming/MatchMessageHandler.cs` — routing and retired-opcode drop
- `client:Application/Match/Outgoing/ClientReady/ClientReadyCommand.cs` — readiness payload
- `client:Game/ScenesV3/ReferenceDuel/ReferenceHud*.cs`, `ReferenceSpellSlot.cs` — HUD rules and keys
- `client:Application/Modules/Spell/Effects/SpellEffectManager.DuelV2.cs` — event → visual
- `client:Tests/DuelV2/*`, `client:Game/ScenesV3/Dev/DuelV2Preview.cs` — validation harnesses

## Combat desktop/mobile profiles (2026-09-09)

See [[combat-ui-profiles]] for the shared profile mechanism, configurable keyboard actions, preview scenes and validation. Desktop uses the bottom action bar and visible keycaps; mobile keeps touch rails. The tutorial follows the chosen profile and respects configured keys and target gates.

## Arena launch/countdown repair (2026-09-10)

`MainReference.tscn` was present. The missing countdown came from server `loading` immediately entering `game_countdown` while the client faded out the lobby and synchronously initialized arena assets. Opcode 16 only raised an event, so values received before HUD subscription were lost.

The active HUD now sends `game_hud_ready` after its first rendered frame and after SceneManager completes the transition. Server loading waits for all human players (bots automatically qualify), then sends opcode 7 and begins the unchanged 2, 1, 0 countdown. Until then the arena shows “Waiting for opponent…”, and combat actions stay gated by the absence of a combat snapshot. Loadout still arrives through opcode 7 → `game_countdown_ready` → opcode 10. No synthetic client-only countdown delays an already running fight.

`MatchContext.PendingGameCountdown` preserves opcode 16 across scene creation; HUD initialization replays it. Zero clears the label; a combat snapshot clears pending countdown and the waiting label even if zero was lost. Reset or a changed match id clears the buffer. Local training is exempt from the waiting message.

Verification: `Dev/DuelLaunchVerification.tscn` replaces the actual lobby through `LobbyStartGameHandler`, injects countdown before HUD creation, checks completed transition/visible countdown/zero clearing, and can capture `/tmp/hexbane-duel-launch.png` with a rendering backend. The regression failed before buffering and passes afterward. Server `phase/loading/phase_test.go` covers slow loading, PvP/AI readiness, irrelevant/invalid/spoofed readiness, full 2/1/0 sequence, and timeout cancellation. `Tests/DuelV2/Live.cs` acknowledges opcode 6 before awaiting opcode 7.

Rollout: server readiness barrier deployed on 2026-09-10 as `sha-b8773ad` (see [[infra-and-deploy]]). Client source is updated locally; rebuild/install clients to include the presentation-ready acknowledgement and buffered countdown. Existing installed applications are unchanged.


## Illustrated duel loading (2026-09-10)

See [[duel-loading-screen]] for the persistent blue/emerald loading illustration, threaded resource progress, one stable random tip, responsive mobile layout and retry/cancellation verification. This user-requested background is a deliberate exception to shared menu artwork. It replaces only the arrangement-to-arena black transition.

## Draft and progression affordances (2026-09-10)

The lobby uses one shared spell detail presenter for library cards and both filled draft rows. Opponent picks carry their server-provided name, icon and description into clickable slots. The Primary character tab uses a compact icon tree with a path switch; dashboard cards pulse while stat, magic or Primary points are available. Menu buttons receive a generated short click through the global `MenuPlayer` binder.
