---
type: project
project: Hexbane
area: audits
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
tags: [hexbane, audit, docs-verification]
---

> Audit report from the 2026-09-07 documentation consolidation (see [[10_Projects/hexbane/_state|_state]] decisions log). Discrepancies here were already applied to the merged notes; the "code smells" and "open questions" sections are the backlog. Paths: `client:` = `~/RiderProjects/hexbane`, `server:` = `~/GolandProjects/hexbane-server`.

# Report: client docs (Godot/C# repo) — 2026-09-07

Staged: `staging/client/{architecture,duel-v2-client,design-system,social-sign-in,tutorial,vfx-and-race-animation,spell-vfx-configuration,deploy-android}.md`. No `staging/plans/create-character-ui.md` (see verdicts). Nothing in either repo was modified.

## 1. Input → verdict

| Input | Verdict |
|---|---|
| `CLAUDE.md` | merged into `architecture`, `deploy-android`, `social-sign-in`, `vfx-and-race-animation` (autoload list, SDK version and effect-type claims corrected) |
| `AGENTS.md` | merged into `architecture` (build/test/conventions); "no test project" claim dropped as stale |
| `docs/DESIGN_SYSTEM.md` | merged into `design-system` (theme path and Marcellus wiring corrected; numeric tables condensed) |
| `docs/client/social-sign-in.md` | merged into `social-sign-in` (accurate; condensed) |
| `docs/client/tutorial-verification.md` | merged into `tutorial` (verification section) |
| `docs/opcodes/duel-v2.md` | merged into `duel-v2-client` (client parts), protocol parts left to `protocol/combat-v2` |
| `docs/opcodes/duel-v2-verification.md` | merged into `duel-v2-client` (results/caveats); changed-file inventory and `/tmp` log references dropped (delivery diary, logs not in repo) |
| `docs/client/reference-duel/README.md` | merged into `vfx-and-race-animation` + `duel-v2-client`; rollback instructions to `backups/…gameplay.zip` dropped (one-off delivery note) |
| `docs/client/arena-maps/README.md` | merged into `vfx-and-race-animation` |
| `docs/client/cast-charge/README.md` | merged into `vfx-and-race-animation` |
| `docs/client/gesture-occlusion/README.md` (PL) | merged into `vfx-and-race-animation` (translated) |
| `docs/client/gesture-vfx/README.md` (PL) | merged into `vfx-and-race-animation` (per-clip GIF table dropped, evidence links) |
| `docs/client/meditation/README.md` | merged into `vfx-and-race-animation` |
| `docs/client/mirror-reflection/README.md` | merged into `vfx-and-race-animation` + `spell-vfx-configuration` |
| PNG/GIF/JSON under `docs/client/*/` (~800 MB) | dropped: not documentation (generated verifier output, see §5) |
| `docs/client/race-selection.md` (not in my list, found in tree) | not merged: it is a server contract copy; **stale** (starter rule "exactly 4 spell_ids", old spell ids `ember_burst`… , stat-scaled preview formulas that duel_v2 disabled). Flag for the protocol/server owner. |
| `docs/opcodes/spell-visual-key.md` (found in tree) | merged into `spell-vfx-configuration` (wire format summary) |
| `docs/Plans/create_character_ui.md` | dropped: stale — targets `Game/ScenesV2/UI/CreateCharacter/*` which no longer exists; the ScenesV3 five-step wizard superseded it. No unimplemented items worth a plan. |
| `docs/superpowers/specs/2026-06-18-race-maker-rework-design.md` | dropped: not client documentation — it is a skill/tooling design (ChatGPT/Photoshop automation) already reflected in the `race-maker` skill; the sprite pipeline it predates is Blender-based now |
| `docs/superpowers/specs/2026-09-06-tutorial-design.md` (PL) | merged into `tutorial` |
| `docs/superpowers/plans/2026-09-06-local-tutorial.md` | merged into `tutorial` (all tasks checked; reward wording superseded) |
| `Resources/SpellVisuals/README.md` (PL) | merged into `spell-vfx-configuration` + dev-scene table |
| `Application/CQRS/README.md` | merged into `architecture` (summary + pointer; the README itself is accurate and generic) |
| `Resources/Races/_tools/blender/README.md` | merged into `vfx-and-race-animation` (summary + pointer; keep the file as the step-by-step, it is accurate) |
| `prompt.md` (root) | dropped: not documentation — a ChatGPT system prompt for spell→JSON generation (`spell.json` schema), part of the old `/old_create_spell` tooling |
| `spell_output.md` (root) | dropped: not documentation — integration guide for the generated "Spell Asset Manifest" (`Resources/Spells/manifest.generated.json`), old spell-generator output format, not used by current code |
| `vfx.md` (root) | dropped: not documentation — ChatGPT prompt for the VFX planning JSON; same old generator |
| `.claude/agents/ui-designer.md`, `.claude/commands/opcodes.md` | not edited; hardcoded paths listed in §4 |

## 2. Discrepancies found (doc said X, code says Y)

1. **Autoloads.** CLAUDE.md lists 6 (DIHost, GameContext, GameEvents, MatchContext, SceneManager, MainThreadInvoker). `project.godot:29-43` has 15: DIHost, DpiScaler, EnvLoader, GameContext, GameEvents, MatchContext, SceneManager, MainThreadInvoker, GodotGTweensContextNode, NotificationManager, MenuPlayer, DevAutoLogin, MCPScreenshot, MCPInputService, MCPGameInspector.
2. **SDK version.** CLAUDE.md "Godot.NET.Sdk 4.5.0"; `hexbane.csproj:1` is `4.5.2` (AGENTS.md is right).
3. **Effect types.** CLAUDE.md lists `Projectile, Static, AreaOfEffect, Beam`; the enum declares all four (`Core/Spells/SpellEffectRegistry.cs:35-44`) but `SpellEffectConfigurations.cs` only uses `Projectile` and `Static` (lines 46-352); no `AreaOfEffect` or `Beam` scene exists.
4. **Spell effect ids.** CLAUDE.md implies spells are configured by spell id; the registry is keyed by 20 legacy FX ids (`heal, fireball, …`), and the 14 server ids reach it only through `SpellEffectManager.DuelV2.VisualId` (`:16-20`) and `Spell.GetIconPath` (`Spell.cs:88-96`).
5. **Game flow.** CLAUDE.md "Game → ArcaneDuel/Main.tscn"; route `normal_game` is `Game/ScenesV3/ReferenceDuel/MainReference.tscn` (`SceneManager.cs:30`). Also the tutorial sits between login and the menu for new accounts (`SceneManager.cs:281-300`), not mentioned anywhere in CLAUDE.md.
6. **Lobby "30s spell selection"** (CLAUDE.md) — no 30 s constant found on the client; timing is server-owned. (unverified)
7. **Dead routes.** `SceneManager.cs:16-17,35` still map `startup`, `autoload`, `debug_rpcs` to `Game/Scenes/…`; the `Game/Scenes` directory does not exist. `vfx_test` has no caller.
8. **Design system theme path.** `docs/DESIGN_SYSTEM.md` says `_Themes/m_charcreate_theme.tres`; actual `Game/ScenesV3/_Themes/`.
9. **Marcellus.** DESIGN_SYSTEM says `SubheadLabel`, `SideTitle`, `StepName`, `TabButton` are Marcellus; `m_charcreate_theme.tres:559-604` wires `Inter_24pt-Medium` for all of them; Marcellus is used only by `SubtitleLabel` in `m_auth_card_theme.tres:201`. `StepCaption` variation exists in the theme but not in the doc.
10. **AGENTS.md "No dedicated automated test project"** — `Tests/DuelV2` and `Tests/Tutorial` exist and are excluded from the main build (`hexbane.csproj:15`).
11. **AGENTS.md "F5 runs from AuthScreen.tscn"** — correct (`project.godot:18`). **AGENTS.md "Consult docs/opcodes/"** — those docs still document retired opcodes 11–15/21–28 as live (`docs/opcodes/op_11…op_28*.md`), while `MatchMessageHandler.cs:23` drops them.
12. **duel-v2.md "Creation … sends spell_ids"** vs **duel-v2-verification.md "Creation skips spell-picking and sends … without spell_ids"** — the later "Starter selection correction" section and code agree with the former: `CreateCharacterCommandHandler.cs:49` sends `spell_ids`; `RequiredStarterSpells` = 3 (+1 Human) (`CreateCharacterScreen.cs:378`).
13. **duel-v2.md validation port** `NAKAMA_URL=http://127.0.0.1:57350` — a disposable stack from that session; `Tests/DuelV2/Live.cs:72` requires whatever `NAKAMA_URL` says; `live_rpc.py` uses `7350`.
14. **cast-charge README "13 clips"** — each race has 16 clips (13 Mixamo + 3 meditation) in `frames.tres`; hand tracks and depth atlases cover the 13.
15. **blender README "on opcode 21 plays the cast clip"** — opcode 21 is retired; casts now come from opcode 32 snapshots / 31 events via `BeginCast`/`ReconcileCast`. Same README: "`adjusted_casting_time` (DEX/race-adjusted)" — stat scaling is inactive in duel_v2. (server-side, unverified here)
16. **race-selection.md** "Exactly 4 starter spells", example ids `ember_burst, flame_orb…`, Human "11 slots instead of 10", stat-based preview formulas — all pre-duel_v2. Client `StatAllocation.BaseSpellSlots = 3` (+1 Human).
17. **social-sign-in.md** is accurate; one nit: it says exchange retries "twice more" — code matches: `ExchangeAttempts = 3`, `ExchangeRetryDelay = 2 s` (`GoogleOAuthSignIn.cs:72-73`). No discrepancy.
18. **SpellVisuals README** says presets are keyed by spell id; existing presets are `fireball.tres` (FX id, server id is `firebolt`) and `magic_reflection.tres` (matches nothing). `magic_arrow`, `mirror_reflection` are correct.
19. **spell-visual-key.md** "Backend work is required" — still true; server does not emit `visual_key` (per protocol staging). Client reads `Spell.VisualKey` (`Spell.cs:71`).
20. **tutorial plan** says "one-time server reward, claim_reward"; superseded by the spec addendum and server rejection; client `TutorialReward` DTO remains.
21. **tutorial-verification.md** command uses `--scene res://…`; `TutorialVerification.cs` sizes: default 1360×768, `phone` 1360×612, `wide` 1560×720, `pc` 1920×1080, `tablet` 1024×768 (doc only mentions phone).
22. **CLAUDE.md `.env` "Requires .env with Nakama server configuration"** — `EnvLoader` only prints an error when missing (`EnvLoader.cs:36`); defaults to `127.0.0.1:7350`, so it is optional.

## 3. Code smells / dead code noticed

- Retired-opcode handler folders still compiled: `Application/Match/Incoming/{GameTimeUpdate, GameplayUpdatePlayers, GameplayLog, GameplayEffectUpdate, GameEffectRemoved, GameplaySpellCasting, GameplaySpellRelease, GameplaySpellFailed, GameplaySpellAccepted, GameplayMeditateAccepted, GameplayMeditateFailed, GameplayMeditateInterrupted}` — unreachable since `MatchMessageHandler.cs:23`. `Opcodes.cs` still declares them.
- `Game/ScenesV3/GameHud/**` (old HUD, `GameHudScreen`, `ArcaneDuel/Main.tscn`, `Main.tscn`) only referenced by `Dev/GameHudPreview`, `Dev/HudPreview`.
- FX/config entries with no server spell: `aqua_pulse, ember_burst, frost_cut, spark, stoneguard, reflection`; scenes `Game/FX/MirrorReflection.tscn`, `Reflection.tscn`.
- `Resources/Spells/` leftovers: `manifest.generated.json`, `spell.json`, `fireball.png` at root, `motions/`, `primitives/`, `references/`, `textures/`, `v2/`.
- `Resources/SpellVisuals/magic_reflection.tres` — no such spell id.
- Two different `export_hand_tracks.py` (`Resources/Races/_tools/` and `_tools/blender/`), `diff` shows they differ.
- `docs/client/reference-duel/` has **no `.gdignore`** and contains 34 `.import` files: Godot imports 53 MB of evidence PNGs into `.godot/imported` and they can reach exports. The other six evidence folders are ignored.
- `SceneManager` dead routes (`startup`, `autoload`, `debug_rpcs`, `vfx_test`) and the commented `character_menu`/`options_menu`.
- `Application/Authentication/Social/PlayGamesSignIn.cs`, `Resources/Images/Auth/icon_play_games.png`, `addons/GodotPlayGameServices` — intentionally retained but unused.
- `Core/Characters/StatAllocation.cs:8` and the auth gateways reference `docs/client/*.md` paths in XML comments; they will dangle once docs move to the vault.
- `docs/Server/**` and `docs/v2/` (empty `server/` dir) are copies outside my scope but still in the repo.
- `project.godot:70-71` carries the Google Desktop client id **and secret** in a committed file (by design per the docs, but worth an explicit decision entry).
- `application/config/features` lists `Mobile`; `.env` has a duplicated `NAKAMA_HOST` key and a `; NAKAMA_HOST` comment line.

## 4. Hardcoded docs paths in `.claude/**` (not edited)

| File | Line | Path |
|---|---|---|
| `.claude/commands/opcodes.md` | 10 | `docs/opcodes/` |
| `.claude/commands/opcodes.md` | 14, 28 | `docs/opcodes/README.md` |
| `.claude/commands/opcodes.md` | 15, 29 | `docs/opcodes/shared_types.md` |
| `.claude/commands/opcodes.md` | 30, 47, 67 | `docs/opcodes/op_XX_name.md` |
| `.claude/agents/ui-designer.md` | 12, 14, 22, 31, 84, 139 | `docs/DESIGN_SYSTEM.md` |
| `.claude/agents/codebase-locator.md` | 62, 89 | generic `README*`, `docs/feature/` (template text, not project-specific) |

Also `CLAUDE.md` itself references `docs/opcodes/`, `docs/client/social-sign-in.md`, `Resources/Races/_tools/blender/README.md`, and the skills `create_spell_*` may reference `Resources/Spells` conventions (not audited).

## 5. C# files writing into `docs/client/`

| File | Line(s) | Target |
|---|---|---|
| `Game/ScenesV3/ReferenceDuel/VerifyPreview.cs` | 15 | `res://docs/client/reference-duel/` |
| `Game/ScenesV3/Dev/ArenaMaps/VerifyArenaMaps.cs` | 14 | `res://docs/client/arena-maps/` |
| `Game/ScenesV3/Dev/ArenaMaps/VerifyRaceCharge.cs` | 19 | `res://docs/client/cast-charge/` |
| `Game/ScenesV3/Dev/ArenaMaps/VerifyGestureVfx.cs` | 32, 77, 120, 157 | `res://docs/client/gesture-vfx/{verification.json, <race>-<clip>.png, layout-*.png, motion-*.png}` |
| `Game/ScenesV3/Dev/ArenaMaps/VerifyGestureOcclusion.cs` | 22, 84, 127, 132 | `res://docs/client/gesture-occlusion/{verification.json, *.png, unmasked-*.png, probe-*.png}` |
| `Game/ScenesV3/Dev/ArenaMaps/VerifyMirrorWard.cs` | 25, 94 | `res://docs/client/mirror-reflection/{<name>.png, burst-NN.png}` |
| `Game/ScenesV3/Dev/MeditationVfxPreview.cs` | 37 | `res://docs/client/meditation/energy-preview.png` (only with `--capture`) |

Comment-only references (no writes): `PlayGamesSignIn.cs:39`, `ISocialAuthGateway.cs:10`, `GoogleAuthGateway.cs:13`, `StatAllocation.cs:8`.

If `docs/` is removed from the repo, these seven writers need a new output root (e.g. a `.gdignore`d `verification/` folder) or the verifier scenes will fail to open the file.

## 6. Open questions

1. Where should verifier evidence go once `docs/` leaves the repo — keep `docs/client/<feature>/` as a gitignored output folder, or move to `verification/`?
2. Should `docs/opcodes/op_11…op_28` and the retired handler folders be deleted now that the dispatcher drops those opcodes?
3. `SceneManager` dead routes (`startup`, `autoload`, `debug_rpcs`, `vfx_test`): delete or restore?
4. Is the Google Desktop client secret in `project.godot` an accepted decision for the vault decisions log?
5. `Resources/SpellVisuals/fireball.tres` vs `firebolt`: which key should presets use (server id per the README)?
6. Is `docs/client/race-selection.md` still wanted as a client note after duel_v2 (its formulas are inactive)?
