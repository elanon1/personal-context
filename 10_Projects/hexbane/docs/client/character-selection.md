---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-17
updated: 2026-09-17
verified: 2026-09-17
tags: [hexbane, client, characters, auth]
sources: ["client:Game/ScenesV3/CharacterSelection/CharacterSelectionScreen.cs", "client:Game/ScenesV3/Auth/LoginPanel.cs", "client:Game/Autoloads/SceneManager.cs", "client:Game/ScenesV3/Dashboard/DashboardScreen.cs", "client:Game/ScenesV3/CreateCharacter/CreateCharacterScreen.cs"]
---

# Character selection (multi-character accounts, HEX-32)

An account owns up to **five** characters; the server keeps exactly one *selected* and every other RPC acts on it ([[rpcs]] → `list_characters` / `select_character`, [[database]] migration 12). The client never sends a character id to progression or spell RPCs to pick a mage — it switches with `select_character` and reloads.

## Flow
- **Sign-in** (`LoginPanel.EnterGame`): `list_characters` → 0 → `SceneManager.GoToCharacterCreation()` (training tutorial, then the wizard); 1 → `GameContext.Character` set, same route (the tutorial screen sends a completed character straight to the dashboard or the progression lesson); ≥2 → `GoToCharacterSelection()`. `DevAutoLogin` mirrors this.
- **Selection screen** (`CharacterSelectionScreen.cs`, built in code on `m_auth_theme`, `ui_background.png`, logo, "CHOOSE YOUR MAGE"): one full-width card per character with the race avatar (`Character.GetRaceAvatarPath`), name, `RaceCatalog.DisplayName` + level, and *PLAY ›*. Footer: `n / 5 characters`, **CREATE CHARACTER** (disabled at the limit), **SIGN OUT** (`GameContext.Logout`). Load failure shows the message plus a RETRY row; an empty roster falls through to creation.
- **Choosing** calls `select_character`, resets `MatchContext`, replaces `GameContext.Character`, clears scene history and enters through the tutorial screen, which skips the arena because `TutorialCompleted` is already true. A busy account (queued or match-locked) gets the server message *Leave the queue or finish your duel before changing character*.
- **Creating another** uses the ordinary five-step wizard; the server marks the new mage selected and `tutorial_completed = true`, so no tutorial runs. The wizard's Back button on step 0 returns to the roster whenever the account already has a character.
- **Dashboard**: LOGOUT was replaced by **CHARACTERS** → selection screen. The only sign-out is on that screen.

## Verification (2026-09-17)
Server behaviour proven over HTTP on the local stack (create second → auto-selected, list 2/5, foreign select refused, switch back changes `get_my_character`/`get_player_spells`, switch and create refused while match-locked) and by the Go integration test (concurrent creates stop at five). Client: build 0 errors, `Tests/JsonAot` context has the new DTOs, `check_aot_json.sh` OK. **Not yet done:** a rendered check of `CharacterSelectionScreen` on a phone-sized viewport and on a device — the screen has no `ResponsiveLayout` compact pass, it relies on a `ScrollContainer`.
