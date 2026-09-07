---
type: project
project: Hexbane
area: infra
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, infra, git, repos, branches]
sources: ["client:git status/log (2026-09-07)", "server:git status/log (2026-09-07)", "vault:10_Projects/hexbane/_state.md"]
---

# Repositories and branches — snapshot 2026-09-07

Both repos are mid-redesign with the whole duel_v2 / spell-system work **uncommitted**. Each checked-out feature branch points at exactly the same commit as its default branch (`git rev-list --count` both directions = 0), so "branch" here is a label only; nothing is pushed. CI and Argo CD only act on `main` (see [[infra-and-deploy]]), so the cluster still runs the pre-redesign server.

## Client — `~/RiderProjects/hexbane`

| Item | Value |
|---|---|
| Remote | `origin` = `git@github.com:elanon1/hexbane.git` |
| Checked-out branch | `feat/duel-v2-client` (= `master` = `01ec454`, no upstream) |
| HEAD | `01ec454` 2026-08-01 "Update to SDK 4.5.2; enhance race-specific assets, login automation, and UI design." |
| Other local branches | `master`, `claude/agitated-lederberg-7214c8`, `claude/cranky-northcutt-2febd8`, `claude/eloquent-swirles-35b374` |
| Remote branches | `origin/master` only |
| Working tree | 593 entries: 96 modified, 95 added (staged), 88 deleted, 5 renamed, 1 renamed+modified, 308 untracked |

Uncommitted work by top-level directory (`git status --short`, entries):

| Dir | Entries | What it is |
|---|---|---|
| `Resources/` | 326 | six race folders + `3d/` + `_tools/` (20 untracked trees, 2 added), new fonts (79 untracked, 48 deleted), `Images/` per-screen assets (29 new, 33 deleted), `Spells/` (19), `Music/`, `Environments/`, `SpellVisuals/` |
| `Game/` | 110 | ScenesV3 rewrites (54 untracked incl. `ReferenceDuel/`, `Dev/`, `Components/`), `FX/` (4), autoloads, 5 deleted scenes |
| `Application/` | 47 | duel_v2 match handlers (`Match/`), `Authentication/` (Google), `Modules/` (character details, tutorial, race), Nakama client |
| `reference/` | 26 | design mock PNGs |
| `Core/` | 24 | `Characters/` (race catalog, stat allocation), `Auth/`, `Spells/`, `Match/`, opcodes |
| `docs/` | 11 | `docs/opcodes/{duel-v2,duel-v2-verification,spell-visual-key}.md`, `docs/Server/`, `docs/client/*` |
| `create/`, `screen/`, `thoughts/` | 12 | scratch dirs (`create/` 10 files), the 2026-08-31 recon note |
| `addons/` | 5 | `hexbane_android` export plugin, Play Games addon disabled |
| `.claude/` | 3 | untracked `race-maker/scripts/*` |
| root | ~12 | `project.godot`, `export_presets.cfg`, `hexbane.csproj`, `.env`, `.gitignore`, `CLAUDE.md`, `deploy.sh`; junk: `hexbane1.apk` (~1.05 GB, untracked), `hexbane.csproj.old.1..5`, `unnamed.jpg`, `.idea/` |

Last 10 commits:
```
01ec454 Update to SDK 4.5.2; enhance race-specific assets, login automation, and UI design.
8b065cd Introduce battle theme and enhance audio handling; adjust music crossfades for match status changes. Disable unused spell effects. Update CreateCharacter screen with race details and stat preview enhancements.
524bace character screeen
30ea610 changes
59915d2 Add avatar and logo assets; include multiple resolutions and SVG versions for logo.
8eaac24 new stuff
3b887c3 JHMHGJJ
cd600bd V0.1
4463164 Introduce authentication and settings features; add `AuthScreen`, `LoginPanel`, `RegisterPanel`, `SettingsScreen`, and related handlers. Update `GameEvents` and integrate progression queries. Cleanup unused folders for consistency.
ef92ce8 Remove unused Arena components (Countdown, HUD elements, SpellButtons, etc.) and their associated scenes and scripts for cleanup and refactoring.
```

Notes:
- `.env` is tracked and modified; it holds the dev login and the Google Desktop-app client secret (names only; see [[infra-and-deploy]] "Secrets").
- `.gitignore` gained `Resources/Races/*/animation/*/render/` and `.mcp_shots/` (working tree). Without the first, a commit of the races would drag ~2 GB of raw renders in.
- Committing the race art as-is adds roughly 6 × (sheets + `.tres`) plus `Resources/Races/3d/` (112 MB of FBX/blend). Whether `3d/` belongs in git is an open question (see report).

## Server — `~/GolandProjects/hexbane-server`

| Item | Value |
|---|---|
| Remote | `origin` = `git@github.com:elanon1/hexbane-server.git` |
| Checked-out branch | `feat/spell-system-redesign` (= `main` = `017ed09`, no upstream) |
| HEAD | `017ed09` 2026-09-05 "fix effect phase" |
| Other branches | local `main`; remote `origin/main`, `origin/cursor/fix-three-codebase-bugs-bf17`, `origin/cursor/manage-persistent-player-effects-b238` |
| Working tree | 276 entries: 97 modified, 123 deleted, 56 untracked |

Uncommitted work by area:

| Dir | Entries | What it is |
|---|---|---|
| `modules/` | 91 | `match/` 43 (duel_v2 engine; deletes `engine/phase/game/{cast_spell,meditate,release_spell,timer_update,update_players,types}.go` and the legacy `engine/player/*`), `spell_system/` 27 (registry, effect handlers, `version.go`, tests), `character/` 9 (details, tutorial, starter spells), `playstyle/` 6, `spellbook/` 5, `main.go` |
| `data/` | 78 | 63 old `spells/<school>/<tier>/*.yaml` deleted, 14 new flat `spells/*.yaml` untracked, `spells - spells.csv` + `spell.example.yml` deleted |
| `docs/` | 43 | `opcodes/` 14 modified (stubs), `match/*` stubs, new `spell_system/{balance-v2,database-v2,verification-v2,spell-lore}.md`, `client/{combat-v2,client-implementation-prompt}.md`, `tutorial.md`, `progression/*` rewritten |
| `db/` | 34 | all 28 tracked migrations (`000001..000009`, `000012..000016`) deleted; new `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial` (6 files) untracked |
| `bkp/` | 18 | old prototype YAMLs deleted |
| `cmd/` | 1 | new `cmd/duel-sim` (untracked) — deterministic duel simulator used by CI's `go test -race` |
| root | 9 | `Makefile`, `docker-compose{,.debug,.prod}.yml`, `Dockerfile.debug`, `RPCs.md`, `.github/workflows/docker-publish.yml`, `scripts/` |

Last 10 commits:
```
017ed09 fix effect phase
2fadd55 feat(spells): standard spells and a 3-6 draft slot ladder
984f391 fix(match): give a new match time to be joined before reclaiming it
274cf50 feat(combat): paralyze cancels a cast in flight
23a3f89 docs(client): race selection contract for character creation
28ad90a docs: rewrite race documentation for the six-race roster
de22c39 feat(character): report race stat limits and signature traits on the sheet
77bdfb3 feat: human spell slot bonus and orc paralyze resistance
936733a feat(combat,match): race damage stat, periodic flat-bonus fix, trait wiring
b3bcda2 feat(combat): per-race dodge scaling and health multiplier
```

Notes:
- The committed history (`b3bcda2..017ed09`, 2026-09-02..05) already contains the six-race roster and race combat traits; the **uncommitted** layer on top is the 2026-09-05/06 spell-system redesign that switched combat to `duel_v2` and made those traits inactive (see [[progression]]). The vault's "35 races" state predates both.
- `.env.dist` and `helm/hexbane/values.yaml` are unchanged in the working tree and still carry the secrets noted in [[infra-and-deploy]].
- `helm/`, `deploy/argocd/` are untouched: the deployment path is still the pre-redesign one.

## Source of truth in code
- `client:.git`, `server:.git` — `git status --short`, `git log --oneline -10`, `git branch -a` as of 2026-09-07
- `client:.gitignore` — what of `Resources/Races` is meant to be tracked
- `server:.github/workflows/docker-publish.yml` — which branch CI builds from
