---
type: project
project: Hexbane
area: audits
status: estimate
created: 2026-09-15
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, platforms, ui, maintenance]
---

# Platform-specific maintenance estimate

Static audit of the current dirty client checkout `47a97c1` and separate web checkout `af927dd`. No game source was modified; no builds or device tests were required for this inventory. Counts are physical lines including comments and whitespace, not executable statements. Estimates describe reviewed branches and responsibilities, not a validated deletion patch.

## Baseline and method

Counted 320 C# files / 31,782 lines under Core, Application, Game, excluding Dev directories and filenames containing Verification, Verify or Preview. This is an approximate application baseline: the heuristic excludes production RaceAnimationPreview (inspected separately), and retains some VfxTest tooling. Core 2,723; Application 9,709; Game 19,350. Third-party plugins, generated code, binaries, assets, scene files, configuration and tests are not in this denominator.

Estimated active compatibility/adaptation responsibility: 1,200–2,000 C# lines (roughly 4–6% of this baseline). Not all can disappear in a single-platform product: screen sizes, resizing and reliable networking remain. Attribution uncertainty is around 30%; this is not a measurement of engineering time saved.

## Concrete evidence

| Area | Measured source footprint | Interpretation |
|---|---:|---|
| Shared menu adaptation | ResponsiveLayout 491, DpiScaler 81, CharacterDetail responsive 82 = 654 lines, plus call sites | Shared between phones, tablets and smaller desktop windows |
| Combat profiles and controls | CombatControls 97, dialog 214, ReferenceHud.Layout 195, CombatSlotGeometry 37, ReferenceSpellSlot 264, TutorialControls 21 = 828 lines | Whole files include shared rendering, persistence and useful controls; do not count all as removable |
| Browser OAuth | GoogleOAuthSignIn 543 | Shared OAuth stays; Android intents, fallback page and iOS return are small portions |
| AOT serialization | ClientJsonContext 105 + SignInJsonContext 18; 63 matching lines across Application | Source-generated metadata is useful beyond iOS; references mostly become simpler, not deleted wholesale |
| AOT tooling | NakamaAot props/XML 18; check_aot_json.sh 28; iOS simulator deployment 113 | 159 support lines; export preset separately |
| AOT tests | JsonAot verification 584 + NakamaAot test 93 | 677 test lines; many validate useful protocol contracts beyond platform constraints |
| Android packaging | deploy.sh 47 + export plugin 46 | 93 support lines; Android and Android Play presets separately |
| Inactive Play Games | PlayGamesSignIn 217; bundled addon 1,453 .gd lines in 22 files | ServiceBootstrapper uses GoogleOAuthSignIn / NullSocialSignIn; inactive provider is cleanup independent of dropping Android |
| Web host | 67 C# boot lines + 325 HTML + 79 project/config + 26 shell = 497 | Mostly scaffold; outside main checkout; SDK/runtime/publish and keeping two code versions aligned are larger risks than line count |

Export preset blocks including options and whitespace: Windows 74, Android 234, macOS 266, iOS 274, Android Play 233. These are configuration, not application code or proportional maintenance costs.

## Removal scenarios

Assume that abandoning a platform also ends its release/testing commitment. Estimates below exclude inactive Play Games cleanup, and are not additive. Web host removal is always separate from the 31.8k main C# baseline.

| Scenario | Estimated removable active C# | UI consequence | Other savings / complexity |
|---|---:|---|---|
| Drop iOS, keep Android + PC + web | 20–80 | Almost none; Android still needs compact UI, touch and safe areas | Remove simulator script 113 lines and iOS preset 274; reduce Apple/AOT-specific validation. Retain generated JSON by default |
| Drop Android, keep iOS + PC + web | 80–180 | Almost none; iOS retains mobile UI | Remove 93 support lines and two presets 467 lines; Android return/back behavior simplifies |
| Drop web | Approximately 0 in main; 67 C# host + 430 other host/support lines externally | None if Android/iOS/PC remain | Separate toolchain, host, browser QA and branch synchronization disappear |
| Drop Windows OR macOS while retaining another PC target | 0–30 | Same desktop UI remains | Remove appropriate preset 74 / 266 lines and corresponding release QA |
| Native PC only, no mobile browser support | 500–1,000 | Remove mobile HUD/gesture variants, safe-area branches and phone-specific compaction tuning; preserve resizable desktop menu behavior | Both mobile release pipelines and web host disappear; largest cross-platform reduction |
| Android only, touch-only product | 400–700 | Remove PC action bar/key labels, keyboard remapping UI and desktop tutorial variant; preserve phone/tablet reflow | Remove iOS + PC releases and web host; controls modal still needs general settings |

Retaining browser support on phones retains mobile input/layout obligations even when native Android/iOS exports disappear. Dropping macOS does not require abandoning the Mac development environment.

## Simplification opportunities without dropping platforms

- Retire the inactive Play Games provider/addon after a focused reference/export audit: candidate 217 C# + 1,453 bundled GDScript lines. This is dead integration cleanup, not savings caused by withdrawing Android.
- Consolidate repeated safe-area conversion in ResponsiveLayout, ReferenceHud, CombatControlsDialog, DuelLoadingScreen and GameOver layout. Benefit is one policy and one place to fix coordinate conversion, not a large line reduction.
- Keep explicit PC/mobile HUD strategies around shared state, casting and slot behavior; avoid splitting whole scenes/game logic.
- Keep JSON metadata and useful contract tests even without iOS unless a separate simplification demonstrably pays off.
- Prioritize decisions by actual release/QA cost, not counts of generated export options. No engineering-hour estimate was measured.

## Source of truth in code

- client:Game/ScenesV3/_Common/ResponsiveLayout.cs
- client:Game/Autoloads/DpiScaler.cs
- client:Game/ScenesV3/CharacterDetail/CharacterDetailScreen.Responsive.cs
- client:Game/ScenesV3/Settings/CombatControls.cs and CombatControlsDialog.cs
- client:Game/ScenesV3/ReferenceDuel/ReferenceHud.Layout.cs, ReferenceHud.cs, ReferenceSpellSlot.cs, CombatSlotGeometry.cs
- client:Game/ScenesV3/Lobby/SpellArrangementView.cs
- client:Game/ScenesV3/Tutorial/TutorialControls.cs
- client:Application/Authentication/Social/GoogleOAuthSignIn.cs and PlayGamesSignIn.cs
- client:Game/DI/ServiceBootstrapper.cs
- client:Application/Serialization/ClientJsonContext.cs and Application/Authentication/SignInJsonContext.cs
- client:Build/NakamaAot.props, Build/NakamaAot.xml, Scripts/check_aot_json.sh
- client:deploy.sh, deploy-ios-simulator.sh, export_presets.cfg
- web checkout:hexbane.web/, Scripts/web.sh, Directory.Build.props, Directory.Build.targets, global.json

