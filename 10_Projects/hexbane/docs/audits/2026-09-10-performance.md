---
type: project
project: Hexbane
area: audit
status: active
created: 2026-09-10
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, performance, mobile, vfx, server]
sources: ["client:Game/ScenesV3/Dev/VfxPerformanceVerification.cs", "server:modules/match/engine/phase/game/performance_test.go"]
---

# Mobile VFX and concurrent combat performance

Reported device: OnePlus 13; stutters occur inconsistently, not exclusively on first casts. The target includes weaker phones. No Android device was attached over ADB during this investigation, so desktop measurements below are not mobile FPS measurements.

## Cleanse portability correction

`SpellField.gdshader` used `pow` with potentially negative bases (including Cleanse's x coordinate and moving sweep). GLSL leaves negative-base `pow` undefined even when the floating-point exponent is an even integer; invalid results can contaminate premultiplied output across the quad. Explicit square/fourth/sixth products now preserve the intended shape with defined arithmetic. The same shared shader also had reversed `smoothstep` edges; these now use one minus an ascending smoothstep. Segment normalization guards coincident endpoints. Pixels outside the already-zero edge mask skip expensive procedural work.

This is a confirmed shader portability defect and a plausible explanation for the reported Android black rectangle, **not an on-device reproduction**. The old and fixed versions both rendered correctly on Apple M3 Pro. GPU checks cover Cleanse at four ages and three scales over a visible background; real arena captures preserve its pale-gold vitality sweep. Neither palette nor gameplay timings changed.

Reference: [Khronos GLSL built-in function contract](https://docs.vulkan.org/glsl/latest/chapters/builtinfunctions.html).

## Client hot-path changes and measurements

Shader uniform names and current animation names use cached `StringName`s instead of allocating native wrappers from strings on every call. Gesture and charge trail scratch arrays are reused; invisible charge/trail effects and idle actor occlusion skip work. Materials still receive changing pose/depth/time data while visible. Spell slots invalidate their drawings only when availability/progress/queued/tutorial-highlight state changes.

Local CPU microbenchmark: macOS, Apple M3 Pro, Godot 4.7 GL Compatibility, 2,000 calls after 100 warmups. This measures C# update calls, not total rendered frame duration or GPU work.

| Update | Before µs/call | After µs/call | Before → after allocated B/call |
|---|---:|---:|---:|
| Spell field | 6.01 | 1.29 | 2,400 → approximately 0 |
| Idle race actor | 30.95 | 0.34 | 5,872 → approximately 0 |
| Casting race actor | 86.01 | 22.91 | 14,640 → approximately 2,037 |

Unchanged spell slots previously emitted 20 drawing callbacks across ten frames (slot plus cast overlay); after the change they emit zero. Allocation-budget checks cover the field/idle/casting updates. Raw measurements and captures live in `client:verification/performance/`; reproduction is `HEXBANE_IGNORE_ENV_FILE=1 <godot> --path . --resolution 1200x540 res://Game/ScenesV3/Dev/VfxPerformanceVerification.tscn -- --tag=current` after a .NET build. GPU mode is required for image/highlight checks.

## Server optimization

Committed as `f64b8fe8306bb60539a5cf974214f8f3b83a8ca5` and deployed successfully; runtime digest and API checks are tracked in [[infra-and-deploy]].

Typed snapshot envelopes/player/action payloads replace transient `map[string]interface{}` construction and JSON map sorting. Opcode 32 fields, explicit nulls, recipient-private queue/sequence, 100 ms simulation ticks and 200 ms snapshot cadence are unchanged. Equivalence tests compare old/new decoded JSON through 40 combat ticks including end state; privacy/null checks and an allocation budget protect the contract.

Go 1.24.5, darwin/arm64, Apple M3 Pro; medians of three one-second benchmark runs:

| Work | Before → after ns/op | Before → after B/op | Allocations before → after |
|---|---:|---:|---:|
| Active snapshot | 3,945 → 1,394 | 5,059 → 2,690 | 75 → 11 |
| Active tick, serial | 5,524 → 2,785 | 6,363 → 3,850 | 95 → 25 |
| Active tick, parallel aggregate | 3,391 → 1,572 | 6,389 → 3,865 | 95 → 25 |

Resident-set benchmark cycles 100 / 1,000 / 10,000 independent active duels with staggered casts and impacts. Three runs of 300,000 aggregate ticks give median 3,137 / 3,132 / 3,435 ns per serial tick, 25 allocations per tick and approximately 13.06 KB of retained fixture heap per duel. Setup is excluded; restarts at 1,200 ticks are included. Ten thousand synthetic states occupy approximately 131 MB of Go fixture heap, not full Nakama RSS.

Reproduce: `go test ./modules/match/engine/phase/game -run '^$' -bench BenchmarkCombat -benchmem -benchtime=1s -count=3`; resident-set variant: `-bench BenchmarkCombatResidentTick -benchtime=300000x -count=3`. Raw results: `/tmp/hexbane-perf-before.txt`, `/tmp/hexbane-perf-after.txt`, `/tmp/hexbane-perf-resident.txt`.

These benchmarks include command submission, casts, impacts, serialization and stub dispatcher calls. They exclude socket transport/command decoding, Nakama scheduling, matchmaking, settlement and database renewal. **They do not establish production capacity of 10,000 concurrent games.** Core RunLoop synchronously renews match locks in the DB every 300 ticks; DB latency and status-heavy Info logging remain candidates for a measured end-to-end load test. Do not add concurrent mutations to an individual match or simply increase replicas without checking Nakama's deployment model.

## Verification and remaining work

- .NET build: 0 errors, 11 existing warnings.
- Full shared spell-field GPU integration: 146 checks, 0 failures (live events, saved-spell preview, all six races/occlusion, VfxTest controls).
- Go full tests, full race tests, vet, allocation/JSON contract tests, parallel and resident benchmarks pass.
- Gesture verifier: 8,899 checks, 0 failures; combat control checks pass, including 30 layout/scale/viewport combinations. GPU tutorial-pulse removal regression failed before the change-detecting setter and passes afterward.
- Gesture occlusion GPU verifier: 12,398 checks, 0 failures (all races, HD/SD, both sides).
- Android debug export completed with exit 0; `apksigner verify` passes and the APK contains the corrected field shader. Artifact: `client:verification/performance/hexbane-performance.apk` (~840 MiB). Godot emitted an EditorSettings/ADB shutdown diagnostic after successful export; signature and archive checks passed. No phone was attached, so no installation or device runtime test was possible.
- Backend CI/deployment passed; live healthcheck and two RPCs return HTTP 200.
- Remaining: physical OnePlus 13 and weaker Android frame-time profiling; real Nakama+DB+network capacity testing on staging.

## Source of truth in code

- client: `Game/FX/SpellField.cs`, `Game/FX/SpellField.gdshader`
- client: `Game/ScenesV3/Components/{VfxUniforms,PoseOcclusion,RaceSpriteAnimator,GestureVfx,CastCharge,MeditationVfx}.cs`
- client: `Game/ScenesV3/ReferenceDuel/ReferenceSpellSlot.cs`
- client: `Game/ScenesV3/Dev/VfxPerformanceVerification.cs`
- server: `modules/match/engine/phase/game/{phase,snapshot,performance_test}.go`
- server: `modules/match/engine/core/loop.go`
