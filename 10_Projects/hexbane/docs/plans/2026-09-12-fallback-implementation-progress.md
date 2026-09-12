---
type: project
project: Hexbane
area: plans
status: active
created: 2026-09-12
updated: 2026-09-12
verified: 2026-09-12
---
# Fallback implementation ledger

Plan: [[2026-09-08-fallback-player-plan]]; spec: [[2026-09-08-fallback-player-design]].

User authorized implementation with 15–30 s fallback. Branch feat/natural-fallback-player in current clean backend checkout. Client has unrelated uncommitted work; preserve it.

Ruling: use 15/22.5/30 s triangular deadline — user changed interval, symmetric mode is a routine choice.
Ruling: preserve current progression2.4 and six redesigned races, ranked built-in queue and existing character.SettleMatch receipt ledger — these supersede the September8 baseline. Do not add obsolete XP/daily logic or duplicate result settlement migrations.
Ruling: implement new queue and AI in independent bounded agent tasks, root integrates lobby/client/lifecycle. Shared source paths explicitly assigned; no commits during concurrent writes.
Ruling: use existing backend checkout on a dedicated feature branch — backend baseline is clean; keep implementation visible in the user's workspace.

| Tasks | Shared contract | Resolution |
|---|---|---|
|1/2/3|allocation → persona → MatchCreate|Allocation ID/generation/humans/fallback/level/seed; root creator integrates|
|2/7|queue DTOs|five RPCs, authenticated identity, generation fencing; root client|
|3/4/5|bot runtime seed/style|MatchState.BotSeed/BotStyle; no wire fields|
|4/5|spell selection → observation|only actual drafted loadout; ownership validation|
|5/6|pure brain → game Submit|immutable public observation; independent RNG|
|7/existing rewards|settlement receipts|reuse current character.SettleMatch|
|1–8|test/migration paths|new migrations000006 queue,000007 personas; isolated SQL test DB|

Tasks1–2: in progress (queue agent).
Task3: in progress (personas agent).
Task4: in progress (root).
Tasks5–6: in progress (combat_ai agent).
Task7: pending (root).
Task8: pending.

## Source of truth in code
- server: modules/match, modules/matchmaking, modules/bot_persona
- client: Application/ArcaneDuel/Normal
