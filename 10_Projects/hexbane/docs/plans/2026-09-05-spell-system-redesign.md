---
type: project
project: Hexbane
area: plans
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, plans, spells, balance]
sources: ["server:docs/superpowers/specs/2026-09-05-spell-system-redesign.md", "server:docs/superpowers/plans/2026-09-05-spell-system-redesign.md", "server:docs/spell_system/balance-v2.md", "server:docs/spell_system/verification-v2.md"]
---

# Spell system redesign: remaining work

The 2026-09-05 spell redesign (catalog `duel_v2`, protocol 2, database baseline, simulator, client migration) is implemented; the rules it produced are documented in [[spell-system]], [[progression]], [[database]] and [[combat-v2]]. Only the items below are still open. Status as of 2026-09-07.

## Open items

| Item | Status | Notes |
|---|---|---|
| `make lint` | not run | `golangci-lint` is not installed locally; CI runs tests only (`server:.github/workflows/docker-publish.yml:26-27`), no lint step. Not a passing result. |
| Human playtests: PC and phone, reconnect, RTT 50/150/250 ms with jitter | not done | Release gate. Automated scenarios and WebSocket smoke tests do not measure touch usability or feel. |
| Verified duel dynamics with comparable players | not done | Release gate: no dominant loops (Arrow + mirror re-cast, healing/meditation loops), no mandatory Cleanse in every viable loadout, a way out of 0 mana under poison. |
| Timeout rate | open | Recorded baseline (`duel_v2.1`, seed 42, 1000 bot duels): 40.4 % timeouts, median 130 s. Bot policies and random loadouts drive this; not evidence about human play. |
| Human 7-slot vs 6-slot | untested separately | Baseline only reports descriptive win counts by loadout size. |
| Re-run baseline for `duel_v2.2` | not done | Values are unchanged from `duel_v2.1` (only creation rules changed), but `balance-v2.json` still says `catalog: duel_v2.1`. `make test-balance` reproduces it. |
| Debug image and Helm deployment of the redesigned backend | not exercised | Compose syntax was checked only. |
| Client display of `nature` / `incantation` | deferred | Server serves the fields; client ignores them ([[spell-lore]]). |

## Decisions to keep

- Target duel length 90–150 s is a playtest outcome, not a timer; the 180 s safety limit yields a draw with no HP-advantage win.
- Infinite-heal concerns are addressed by tuning costs/effectiveness first; an overtime mechanic is designed only if results justify it.
- Any return of stat/skill/race combat bonuses needs its own design and balance pass; the old formulas in `modules/combat` must not be re-wired implicitly.

## Source of truth in code

- `server:cmd/duel-sim/main.go`, `server:Makefile` (`test-balance`) — baseline reproduction
- `server:modules/spell_system/version.go` — catalog version to bump after any balance change
