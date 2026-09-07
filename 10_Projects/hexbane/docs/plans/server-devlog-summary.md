---
type: project
project: Hexbane
area: plans
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, plans, worklog, server]
sources: ["server:docs/devlog.md", "server:git log", "server:docs/spell_system/verification-v2.md", "server:docs/superpowers/specs/*"]
---

# Server work log summary

Dated milestones, newest first, one line each. Entries up to 2025-12-31 come from `server:docs/devlog.md` (originally in Polish, condensed and translated; the devlog mixes client and server work). Entries from 2026-03 on come from `git log --oneline` of `hexbane-server`, which has no devlog entries after 2025-12-31. The open TODO/NEXT lists from the devlog are at the end. Current architecture: [[server-architecture]]; workflow: [[dev-setup]].

## 2026 (git history and working tree)

| Date | Milestone | Ref |
|---|---|---|
| 2026-09-05…07 | **Spell system redesign, uncommitted** on the working tree: `duel_v2` ruleset, protocol 2, opcodes 29–32, command queue with `client_seq`, logical 100 ms clock, deterministic effect queue, 14-spell flat YAML catalog (`data/spells/*.yaml`), fixed 200 HP / 100 mana, stat/skill scaling disabled, old opcodes 11–15 and 21–28 removed with `cast_spell.go`/`meditate.go`/`release_spell.go`, migrations squashed to `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial`, `account_tutorials` table + tutorial RPCs, test accounts and seed/reset/clear scripts, `cmd/duel-sim` simulator, `scripts/test_combat_runtime.mjs`, CI now runs race-detector tests; verification in `docs/spell_system/verification-v2.md` | working tree, `docs/superpowers/specs/2026-09-05-spell-system-redesign.md` |
| 2026-09-05 | Fix effect phase | `017ed09` |
| 2026-09-04 | Standard spells (Magic Arrow, Mirror Reflection carried by everyone) and the 3→6 draft slot ladder (Human 4→7) | `2fadd55` |
| 2026-09-04 | Google sign-in design (Nakama built-in `AuthenticateGoogle`, desktop OAuth client, no custom RPC) | `docs/superpowers/specs/2026-09-04-google-auth-design.md` |
| 2026-09-03 | New match waits 30 s for its first player before being reclaimed (`JoinGraceTicks`) | `984f391` |
| 2026-09-03 | Paralyze cancels a cast in flight (mana not refunded) | `274cf50` |
| 2026-09-02 | Race redesign: six-race roster (human, elf, dark_elf, shadow, gnome, orc) with effective-stat limits and typed signature traits; loaded from DB; enforced at creation and level-up; Human spell-slot bonus, Orc paralyze resistance; race docs rewritten | `88bca8a`…`23a3f89` |
| 2026-06-07 | Fix effect phase | `f7065fe` |
| 2026-04-24 | Race element bonuses (primary/secondary affinity damage scaling; later superseded by the 2026-09 roster) | `d3ecf2b` |
| 2026-04-10 | golang-migrate added to the image and Helm chart; Postgres 17.6-alpine; Helm image pull secrets | `ae229d6`, `47d147d` |
| 2026-04-10 | Old spell catalog replaced by Toxic and Earth spells with CSV definitions (superseded in 2026-09) | `23c193f` |
| 2026-03-13 | Subagent definition files, spell attribute updates, casting log improvements | `3e0cfb3` |

## 2025 (devlog)

| Date | Milestone |
|---|---|
| 2025-12-31 | Spell button disabled states, effect icons, meditate button, UI scaling fixes |
| 2025-12-30 | Player health/mana handling; server hosted on a Synology NAS |
| 2025-12-30 | Lobby: turn-based spell selection enforced, spellbook refresh, readiness logging (`10f1819`) |
| 2025-12-12 | New spell/health UI, spell buttons |
| 2025-12-03 | Logo image, art direction change, game scene work started |
| 2025-12-01 | Client-side drafting, turn indicator, lobby countdowns |
| 2025-11-27 | Fixed spell duplication after pick, spellbook highlight bug, draft id condition, PvP join fix |
| 2025-11-02 | Lobby started; available spells exposed from backend as opcode 70 |
| 2025-09-09 | Character bar, button/panel visibility, login fixes |
| 2025-09-08 | Initial news list and character bar UI |
| 2025-09-07 | Return after a long break; Play modal with three game modes |
| 2025-08-23 | Bot match with basic situational AI; match module split into `normal_match` / `ai_match` / `engine` (`6e1afdb`); GitHub Actions Docker publish (`e99b243`) |
| 2025-08-22 | Cast/throw animation PoC, key-spam fix, button labels, export library |
| 2025-08-19 | Spell effect engine, Heal / Great Heal PoC |
| 2025-08-18 | Effect UI updates on removal, paralyze effect, shield bug fix; snapshot `IsEqual` with shield and effects (`8c22d9e`) |
| 2025-08-14…17 | Complete rebuild of game mechanics, idle animation, spell visuals |
| 2025-08-14 | `endless_story` module removed (`ce65ca0`); a `create_character` match variant later returned under the same package |
| 2025-08-12 | Character creation reworked from RPCs into a single-player guided match |
| 2025-08-10 | Menu UI finished, match start; AI-assisted character creation refactor begun |
| 2025-08-09 | Full menu UI refactor; client services for Nakama data |
| 2025-08-07…08 | Friends and notifications rebuilt from scratch on both sides |
| 2025-08-06 | Client notification manager/module |
| 2025-08-02 | Social functions skeleton on the server |
| 2025-07-31 | Endless Story PoC (AI story generation) |
| 2025-07-30 | Login fixes; basic server-disconnected handling |
| 2025-07-29 | Mutexes in `PlayerState` against race conditions; GameOver phase, winner evaluation, `TerminateMatch` flag, match restart (`1c7ce74`, `f0e33b1`) |
| 2025-07-28 | Meditation: accepted/failed/interrupted events, mana regeneration |
| 2025-07-27 | Spell system rewritten with OnStart/OnTick/OnEnd; effect icons with timers; no stacking, refresh on reapply; first client animations |
| 2025-07-26 | Animated HP/mana bars, gameplay log, in-game console; one YAML file per spell |
| 2025-07-24…25 | Casting time, spell release flow, player sync (`15254fa`, `4efcc66`); client GameEntrypoint |
| 2025-07-23 | Mana system, spell casting and damage on the server (`a3d3bf9`) |
| 2025-07-22 | Match timer; player identification fix; phase bug pass after protocol change |
| 2025-07-19 | Client/server handshake unified: client ready first, server confirms (`f0a7c58`) |
| 2025-07-18 | GameCountdown phase; global `TicksPerSecond` (`89008a9` phase refactor) |
| 2025-07-17 | Lobby slot limit, Lobby → LobbyCountdown (5 s) → Loading → GameCountdown chain |
| 2025-07-16 | Slot counts in the model, empty slots UI, `MainThreadInvoker`, C# events instead of Godot signals |
| 2025-07-15 | Draft turn id sent in lobby update, alternating spell picking, icon/name loading |
| 2025-07-14 | Connecting phase finished; lobby switch and countdown on the server |
| 2025-07-12 | `MatchState` refactor started; exit/quit buttons |
| 2025-07-11 | Signal loop bug found (deserialization emitted signals); player died and game over modal |
| 2025-07-08 | Spell cast RPC on client and server, mana deduction, `EffectContext` logger |
| 2025-07-07 | Bare spell engine on the server matching the Miro design |
| 2025-07-06 | Move into the match after spell selection; spell system refactor to YAML (`ecefb63`) |
| 2025-07-04 | Client restructure (Core → Application), `EnvLoader`, lobby view with timer and spell selection |
| 2025-07-03 | Main menu, matchmaking, match dispatcher |
| 2025-07-02 | Registration fix, RPC debug view, character creation scene wired to RPCs |
| 2025-06-27 | Character creation RPC and learned-spells RPC |
| 2025-06-26 | Client rewritten from GDScript to C#; Nakama client, login, DI + CQRS singletons |
| 2025-06-24 | First server commit (`c758d73`) |

## Open items carried in the devlog (as written, 2025-12-31)

TODO: endless story PvM, full HP only after the countdown, `OnMatchPresence` / `OnServerError` handling, spell delay bug, server-disconnected handling, statistics, log. NEXT: poison, out-of-mana feedback, enemy health/mana, active effects and timers, sound while music plays, phase game, audio mute option. Several of these are addressed by the 2026-09 redesign (poison, effects, mana feedback via opcode 30/31); the rest are unverified.

## Source of truth in code

- `server:docs/devlog.md` — original Polish devlog (last entry 2025-12-31)
- `server:.git` — `git log --date=short --oneline` for 2026 milestones
- `server:docs/superpowers/specs/*.md`, `server:docs/spell_system/verification-v2.md` — design and verification records of the 2026-09 redesign
