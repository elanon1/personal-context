---
type: project
project: Hexbane
area: audit
status: active
created: 2026-09-16
updated: 2026-09-16
verified: 2026-09-16
tags: [hexbane, server, performance]
---

# Resource cost of one combat

Local measurement: Go 1.24.5, darwin/arm64, Apple M3 Pro, HEAD f05eabd with pre-existing uncommitted changes. Production image at observation: ghcr.io/elanon1/hexbane-server:sha-f05eabd. Local working tree and deployed code are not identical.

Command: `go test ./modules/match/engine/phase/game -run '^TestCombatSnapshot' -bench 'BenchmarkCombat' -benchmem -benchtime=1s -count=3` (PASS).

Three-run median: snapshot 1304 ns/op, 2690 B/op, 11 allocations; serial active tick 3614 ns/op, 11764 B/op, 27 allocations. Resident scenarios: 100 matches 4266 ns/op and 21263 retained B/match; 1000 matches 4156 ns/op and 21230 B/match; 10000 matches 3892 ns/op and 21225 B/match. Retained memory is measured after a short staggered warmup, not peak memory across a full match. The fixture has two human presences, a synthetic damage spell and artificially high health/mana; it does not measure bot AI or a representative spell mix.

At 10 ticks/s, serial timing implies approximately 36 microseconds of measured plugin execution per second of combat (0.0036% of one local core). Extrapolated 180-second combat: 6.5 ms of plugin execution and 20.2 MiB cumulative heap allocations. Allocated bytes are temporary churn, not simultaneously retained RAM. This is not a measured end-to-end CPU bill.

A temporary Go overlay probe ran all 1800 ticks through GamePhaseState.Tick, submitting a cast from each player every 10 ticks. It counted each broadcast once per recipient (nil recipient list = both players): 2,203,093 payload bytes, 6714 recipient deliveries, final Over=true. About 2.10 MiB per 180 seconds, or 12.0 KiB/s total for both clients. Excludes incoming commands, Nakama envelopes/base64 if used, WebSocket/TLS/TCP framing, lobby and game-over reward persistence. Probe source and overlay remained in the temporary directory recorded at /tmp/hexbane-cost-probe-path; no repository source was edited.

Production read-only `kubectl top pods` sample: Hexbane 1m CPU / 32 MiB working memory; PostgreSQL 17m / 41 MiB. These are whole-service observations with unknown active match count, not incremental per-match measurements and not an established idle baseline. Deployment requests 100m CPU / 256 MiB and limits 500m / 512 MiB; reservations/limits are not measured usage.

No production matches were generated. Full per-match server cost remains unmeasured: requires controlled baseline versus completed matches on matching server hardware, including Nakama scheduling/transport, SQL, AI scenarios, GC and peak memory. These results cannot establish production concurrent-player capacity or financial cost per match.

## Source of truth in code

- server:modules/match/engine/phase/game/performance_test.go
- server:modules/match/engine/phase/game/phase.go
- server:modules/match/engine/state/state.go
- server:helm/hexbane/values.yaml


## Hosting planning estimate (same-session follow-up)

Live node sample: 10 vCPU, 16324156 KiB capacity, 453m CPU usage, 7981 MiB working memory. Existing node has headroom at the observation instant; other workloads and CPU sharing remain unknown. Recommend using it for initial playtests, then a separate x86-64 VM for isolation when needed: 2 dedicated vCPU / 8 GB RAM, or 4 shared vCPU / 8 GB. One Nakama instance plus PostgreSQL and TLS ingress, external database backups. No automatic infrastructure changes made.

Unvalidated planning envelope for a VM reserved for the game: 100–300 simultaneous PvP matches on 2 dedicated vCPU / 8 GB, initial launch target 100 after load-test validation. This is an engineering assumption, not measured capacity. A sensitivity allowance of 30–100 times local plugin execution implies roughly 0.108–0.361 CPU cores for 100 matches and 0.325–1.084 cores for 300 matches, before separately budgeted service baseline and headroom. The multiplier is hypothetical and does not establish an upper bound, especially for AI and database bursts. Validate 25/50/100/200/300 matches, including lobby, results and AI, with tick scheduling delay, CPU throttling, RAM and SQL latency. Target p99 match-loop execution below 20 ms within 100 ms ticks, no sustained tick lag, CPU below 60–70%, RAM below 70–75%.

Budget network at an assumed 20–30 KiB/s per match including transport allowance: 100 matches approximately 16–25 Mbps and 5.3–8.0 TB per 30 days if continuously occupied. Real spell mix may differ. Downloadable game assets should be served separately.

Hetzner official June 2026 rate card for Germany/Finland lists CCX13 EUR42.99/month, CX33 EUR8.49/month, excluding VAT/IPv4. CX33 product page showed unavailable; purchase availability requires console verification. CCX13 candidate specs: 2 dedicated vCPU, 8 GB RAM. Cost-optimized CX33: 4 shared vCPU, 8 GB, 80 GB NVMe. Prices exclude backup budget.

Sources checked: https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/ ; https://www.hetzner.com/cloud/cost-optimized/ ; https://docs.hetzner.com/cloud/servers/faq/
