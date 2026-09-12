---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, server, progression, races, stats, skills, combat]
sources: ["server:docs/progression/overview.md", "server:docs/progression/race.md", "server:docs/progression/stats.md", "server:docs/progression/skills.md", "server:docs/progression/progression.md", "server:docs/progression/combat.md", "server:docs/progression/match-integration.md", "server:docs/progression/modifiers.md", "server:docs/superpowers/specs/2026-09-02-race-system-redesign-design.md", "server:docs/superpowers/plans/2026-09-02-race-system-redesign.md", "client:docs/Server/progression/overview.md", "client:docs/Server/progression/race.md", "client:docs/Server/progression/stats.md", "client:docs/Server/progression/skills.md", "client:docs/Server/progression/progression.md", "client:docs/Server/progression/combat.md", "client:docs/Server/progression/match-integration.md", "client:docs/Server/progression/races_seed.sql"]
---

# Character progression — duel_v2.4

User-approved rules implemented on 2026-09-08. This note describes checked-in/workspace code and migrations, not a deployed database. Protocol remains 2. Combat calculations are in [[combat-stat-rules]], primary graphs in [[spell-system]], and requests in [[rpcs]].

## Shared stats and races

Every race has 400 base points at creation and earns 5 per character level, for 545 at level 30. Each stat must be at least 10. There are no racial stat ceilings, unequal flat stat grants, or forced race presets. Base and effective stats are equal under migration 000005. Creation requires exactly 400 points; incremental allocation may spend a subset of unspent points. Free `respec_stats` redistributes the entire earned budget outside a match, then sets unspent points to zero. Build mutations lock the character row and reject an active match lease (code 9).

| Race | Active identity after migration 000005 |
|---|---|
| Human | +1 optional spell slot, including at creation and cap |
| Elf | mana regeneration ×1.2 |
| Dark Elf | poison and delayed-hex damage ×1.1 |
| Shadow | dodge .035% per softened DEX, cap 15%; casting modifier −2 percentage points |
| Gnome | mana costs ×.85, rounded up |
| Orc | HP ×1.05; paralysis duration ×.75; spell power still uses INT |

All six have flat stat modifiers 0, shared minima 10, maxima 0 (unbounded by race), school bonuses 0 and empty per-spell resistance maps. Traits are decoded/validated at startup. Lore/nature does not supply hidden mechanical school bonuses.

## XP and character levels

Cumulative threshold to reach level L is `T(L)=45*(L-1)+12*(L-1)*(L-2)/2`, clamped to levels 1–30. The next step costs 45 XP initially and 381 for 29→30. XP is cumulative; settlement clamps stored character XP at 6177. Remaining XP is `max(0,T(L+1)-XP)`, zero at cap.

| Level | Total XP |
|---|---:|
| 1 | 0 |
| 2 | 45 |
| 7 | 450 |
| 11 | 990 |
| 16 | 1935 |
| 23 | 3762 |
| 30 | 6177 |

A completed win pays 120 XP, loss/draw 70; daily and first-win XP extras are disabled. A first win reaches level 3, a first loss level 2; both earn 5 MP. At an even win rate, 95 XP/match implies about 65 matches to cap, not a match-count gate.

## Magic Points and skills

| Reached levels | MP per level | Cumulative checkpoint |
|---|---:|---:|
| 2, 4, 6 | 5 | 15 by level 7 |
| 8–11 | 5 | 35 by 11 |
| 12–16 | 5 | 60 by 16 |
| 17–30 | 2 | 88 by 30 |

Other levels grant no MP. Available MP remains earned minus spent; learning a spell charges its explicit price, currently 5 for each of 12 optional spells. Ownership is not limited by draft slots and purchases are preserved by migration. Collection targets of 15/16 optional spells require future content; the present optional catalog has only 12.

At character cap, continued result XP fills `study_xp` (0–499): each 500 grants 5 MP and carries overflow. On the match reaching cap only XP beyond 6177 enters study; the final level's MP still pays. No extra stat points or levels are granted. Multiple study thresholds can pay in one atomic settlement, even when every skill is capped.

Meditation, Spell Resistance and Magery each retain their individual 100 cap. They keep growing after character level 30. Each attained 25/50/75/100 threshold grants 5 MP once (maximum 60 across all three skills). `skill_mp_mask` uses bits 0–3 Meditation, 4–7 Spell Resistance, 8–11 Magery, in ascending threshold order. Reset/respec must not clear the mask. Skill training is bounded per action and at +5 per skill per match; see [[combat-stat-rules]].

## Draft slots and primary entitlement

| Character level | Other races | Human |
|---|---:|---:|
| 1 | 3 | 4 |
| 7 | 4 | 5 |
| 11 | 5 | 6 |
| 16–30 | 6 | 7 |

Previously earned higher slots are grandfathered. Match setup clamps usable picks to owned optional spells while exposing full entitlement. The two permanent primary spells are carried on top and cost no optional slots.

Each primary independently earns tiers 1–6 at character levels 1/5/10/16/23/30. Tier entitlement is not an automatically allocated path: empty saved paths resolve to the free root. `set_primary_path` permits a legal contiguous prefix up to the earned tier. Bonuses accumulate through reconnecting graph branches; unspent tiers supply no effect. Graph version is 1. See [[spell-system]].

## Ranked access

The normal matchmaker accepts string property `queue=normal|ranked` (omitted means normal). The server replaces the query with `+properties.queue:<queue>`, forces two distinct players, rejects mixed queues and checks ranked character level ≥30 at queueing, match creation and invited join. No skill, MP, collection or primary-allocation requirement exists. Ranked matches use the same combat/economy; this implements an access gate and separate queue, **not MMR, rating, seasons or leaderboards**.

## Settlement and migration operations

`character.SettleMatch` locks the character, applies XP/level/MP/skills/record changes and writes a `(character_id,match_id)` receipt atomically. Repeated delivery returns the original receipt. Game over persists both human results before broadcasting, so partial database success is retryable without double payment. Bots have no persistence. Existing completed AI duels also use this settlement; private/custom reward modes are not separately implemented. Queueing, lobby cancellation and insufficient-player exits do not themselves settle a reward; only the completed game-over path does. Skills and post-cap MP remain independent of ranked access.

Migration **000004** snapshots existing character rows into `progression_redesign_backup`, preserves earned level and the bounded fraction within the old exponential XP interval, maps capped characters to 6177, and initializes study remainder to 0. It resets base/effective stats to 133/134/133 with `5*(level-1)` unspent points; the free full-budget respec is the reallocation flow. It preserves skills, previous MP/spending and spell ownership, grants existing skill milestones once and records their mask, and preserves the greater of old/new slot entitlement. It adds versioned primary paths, reward receipts and 15-minute match leases. Migration **000005** replaces the six race definitions with the shared-stat trait model. Apply migrations and compatible server/client together when rollout is explicitly performed; no live deployment was performed in this implementation task.

## Source of truth in code

- `server:modules/progression/{constants,xp,magic_points,spell_slots,rewards}.go`
- `server:modules/character/{character,validate,details,rpc,primary_progression,rewards,db}.go`
- `server:modules/primary/graph.go`
- `server:db/migrations/000004_progression_redesign.up.sql`, `000005_race_redesign.up.sql`
- `server:modules/match/normal_match/{matchmaker,join}.go`
- `server:modules/match/engine/{core/player_setup,phase/gameover/phase}.go`

## Preserving active match leases (2026-09-12)

`character.CheckMatchAvailability` performs a read-only check before a new normal/AI join. An unexpired lease for another match returns `ErrCharacterInMatch` (`character_in_match`). `AcquireForMatch` retains the atomic guard and returns the same sentinel. The 15-minute lease, renewal and settlement/release rules are unchanged. There is no orphan-lock reclamation based on whether a match exists in the current process; after a restart the player must wait for expiry if normal release did not happen. Same-match reconnect is permitted.

Regression `TestMatchAvailabilityPreservesExistingLease` checks new-match refusal, same-match admission, unchanged existing lease and expiry in an isolated PostgreSQL test schema.
