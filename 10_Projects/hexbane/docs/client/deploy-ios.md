---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-08
updated: 2026-09-14
verified: 2026-09-14
tags: [hexbane, client, ios, deploy]
sources: ["client:export_presets.cfg", "client:hexbane.csproj"]
---

# Exporting the client to iOS

Use the Godot 4.7 .NET editor and the **iOS** export preset. The export path is
`../export/ios/hexbane.ipa`; create the output directory before exporting.
The preset enables **Export Project Only** for the requested iPhone simulator workflow,
so Godot generates `hexbane.xcodeproj` without attempting a signed device IPA.
The preset excludes HD race sheets and `addons/rider-plugin/*`, whose native libraries
support desktop editor platforms only. Rider remains available in the editor.

Avoid extra dots in the export basename: `apk.app.ipa` generated a project referring to
`apk/dylibs/ExportDebug/hexbane_aot.xcframework` although the framework was actually under
`apk.app/dylibs/ExportDebug/`. Exporting as `hexbane.ipa` produces matching paths.

## iPhone simulator

From the client repository root:

```bash
./deploy-ios-simulator.sh
```

The script targets Godot **4.7 .NET** on Apple Silicon. First run downloads the
matching Godot source from the official GitHub tag, installs SCons into a private Python
venv, and builds a debug ARM64 simulator engine with Mono enabled. Metal and Vulkan are
disabled for this simulator template; Hexbane already uses the GL Compatibility renderer.
The resulting static library is cached under
`~/Library/Caches/hexbane/ios-simulator-4.7/` for subsequent builds.

The script exports the iOS Xcode project, adds the missing ARM64 slice to its engine
XCFramework, builds with `CODE_SIGNING_ALLOWED=NO`, installs on an available iPhone
simulator (preferring an already booted one), and launches the bundle identifier read from the built app Info.plist.
No physical device, Apple Developer profile, or signed IPA is needed. Set iOS
`application/code_sign_identity_debug="-"` for ad-hoc library signing during the Godot
export; otherwise Godot still invokes Apple Development signing even with Export Project
Only and may block on a keychain prompt.

Output: `../export/ios-simulator/`, including `hexbane.xcodeproj`, `export.log`,
`build.log`, and `DerivedData/Build/Products/Debug-iphonesimulator/hexbane.app`.
Engine compilation output: cache `engine-build.log`.

Overrides: `GODOT`, `SIMULATOR_ID`, `HEXBANE_IOS_OUTPUT`, `HEXBANE_IOS_CACHE`,
`HEXBANE_BUILD_JOBS` (default 8). List simulators with `xcrun simctl list devices available`.
Keep output outside the Godot project so generated assets are not scanned into the game.

### Why the ordinary export failed

- The stock Godot 4.5.2 and 4.7 .NET simulator `libgodot.a` is x86_64-only despite its
  XCFramework plist declaring both ARM64 and Intel. The C# AOT framework has both.
- Installed iOS 26.5 simulator accepts ARM64 only. Intel compilation passed but installation
  and explicit Intel simulator boot were rejected. Disabling ARM64 cannot fix this.
- The ARM64 template must be compiled from matching source; the script supplies it without
  replacing globally installed templates. The source version must match the project SDK.
- The open Godot editor overwrote initial external preset edits with its old in-memory
  values. Restart the editor after external edits. The script refuses a stale preset.

Verified 2026-09-09 on Godot 4.7: full source build succeeded (~5.5 minutes), Xcode ARM64 simulator build
succeeded, app installed and launched on iPhone 17 Pro / iOS 26.5, and the Hexbane
authentication screen rendered. Full login and network gameplay were not validated.

## Authentication runtime requirements (2026-09-13)

Keep `Build/NakamaAot.props` and its descriptor in the project: Nakama reflection needs explicit preservation under iOS Native AOT. Post-login bootstrap/tutorial JSON uses generated metadata; since 2026-09-14 every other shipped JSON path does too (next section). The iOS preset also registers the `hexbane` URL scheme to return from browser authentication; see [[social-sign-in]] for the diagnosis, regression tests and validation limits.

`deploy-ios-simulator.sh` reads the bundle id from the built Info.plist instead of assuming the earlier `com.hexbane.game` id. Syntax checked after this change; the full export script was not rerun. The 2026-09-13 auth diagnostic used the earlier simulator app/resources with a freshly published ARM64 C# framework and matching scheme entry. It is a simulator diagnostic, not a signed device release.

## Runtime rule: generated JSON only (2026-09-14)

Symptom on the phone after the 2026-09-13 login fix: character creation failed, the tutorial arena showed no race sprites, and every later RPC/socket screen would have failed the same way. Reproduced in the ARM64 iPhone 17 Pro / iOS 26.5 simulator with the current tree: unified log `InvalidOperationException: Reflection-based serialization has been disabled for this application` from `RaceSpriteAnimator.Load` (hand tracks) and, per the AOT analyzer, from ~30 more `JsonSerializer` calls (`CreateCharacterCommandHandler`, spells, social, notifications, `MatchMessageHandler`, `DuelProtocol`, queue, game over). Cause: iOS is Native AOT, which disables reflection-based System.Text.Json; only the sign-in bootstrap used generated metadata.

Fix: generated contexts for every shipped path (`ClientJsonContext`, `GameJsonContext`, existing `SignInJsonContext`) and named request DTOs instead of anonymous objects — the rule and the type list are in [[client-architecture]] → *JSON serialization*. Nothing changed on the wire.

Verification recipe (all run 2026-09-14):

1. `Scripts/check_aot_json.sh` — AOT/trim analyzer build; fails on any `IL3050` (reflection serializer) outside dev-only scenes. Before the fix: 73 `IL3050` sites, ~30 in shipped code; after: none.
2. `Tests/JsonAot` — Godot headless project that disables JSON reflection through the AppContext switch (Godot ignores the game's `runtimeconfig.json`) and drives the real handlers through the Nakama client with a canned `IHttpAdapter`. Before the fix: 1 passed / 19 failed with the iOS exception; after: 157 passed.
3. `Tests/Auth` still passes (30 checks) — sign-in untouched.
4. Simulator: `./deploy-ios-simulator.sh`, then an automated pass through registration → character creation → post-login screens using the dev auto-login. `EnvLoader` reads process environment variables, and `simctl` forwards `SIMCTL_CHILD_*` to the app:

```bash
SIMCTL_CHILD_DEV_AUTO_LOGIN=true SIMCTL_CHILD_DEV_AUTO_LOGIN_MODE=new_character \
SIMCTL_CHILD_NAKAMA_SERVER=local SIMCTL_CHILD_NAKAMA_HOST=127.0.0.1 \
xcrun simctl launch --console-pty --terminate-running-process <udid> com.dev.hexbane
```

Logging: Godot on iOS writes `GD.Print` to the unified log at **info** level and `GD.PrintErr`/exceptions at error level; `simctl launch --stdout=/path` created no file and `--console` / `--console-pty` showed only Objective-C runtime lines. Read the app's output with `xcrun simctl spawn <udid> log show --info --last 5m --predicate 'process == "hexbane"' --style compact` (drop `--info` for errors only). The local stack must be up (`docker compose up -d` in the server repo; the simulator reaches the Mac's loopback). Evidence 2026-09-14: `[DevAutoLogin] new_character: creating 'Dev3912' race=human …` → `connected (mode=new_character, hasCharacter=True)` → `TutorialScreen`, character row present in the local database, both race sprites visible in the tutorial arena, no serialization errors in the log.

`deploy-ios-simulator.sh` now really reads the bundle id from the built `Info.plist` (the 2026-09-13 note described this, but the script still launched the hard-coded `com.hexbane.game`, i.e. the stale 2026-09-09 install); that stale app was uninstalled from the simulator.

### Second root cause on the same day: C# `dynamic`

With every JSON path migrated, the bot duel still failed on iOS: **Matchmaking — Object reference not set to an instance of an object** right after tapping *vs AI*, while the server log showed the match created (`Match started` → `Nobody joined within the grace period`). `Tests/NakamaAot live-socket` (real Native AOT on macOS) proved the socket RPC envelope is intact, which ruled out the SDK; the remaining `IL2026` in shipped code was `SignalUtils.EmitSafe`'s `dynamic` argument conversion, called by `MatchState.ChangeStatus` for `MatchFound`. `Tests/NakamaAot dynamic-check` reproduces the binder failure natively. Fix: explicit `Variant` conversion (see [[client-architecture]] → *iOS Native AOT rules*); `Scripts/check_aot_json.sh` now also fails on `dynamic`.

## Development signing (physical devices only)

Turn off **Export Project Only** and restore Debug Code Sign Identity to blank /
`Apple Development` when a signed device IPA is wanted.

The current preset bundle identifier is `com.dev.hexbane`. Xcode needs an Apple Development certificate,
the selected team, and a development provisioning profile covering a registered device.
Connect an iPhone/iPad, open the generated `hexbane.xcodeproj`, select the target's
Signing & Capabilities, enable automatic signing, select the intended team and device,
and let Xcode register the device and create the profile. Device registration may require
Developer Mode/trust confirmation on the device or team administrator access.

On 2026-09-08, the .NET export generated its AOT framework and the corrected export reached
Xcode, but Apple rejected profile creation: **Your team has no devices from which to generate
a provisioning profile**. An Apple Development signing identity was present. A signed IPA
has not been verified; registering a device and retrying remains necessary.

Godot's final `Failed to run xcodebuild with code 0` is not a useful root cause: inspect the
preceding Xcode output. Diagnostic logs from this session are at `/tmp/hexbane-ios-export.log`
and `/tmp/hexbane-ios-unsigned.log` (temporary files).

## Validation

The corrected project archived successfully with `CODE_SIGNING_ALLOWED=NO` on Xcode 26.6.
This verifies compilation/linking and archive generation, not device installation or signing.
The unsigned diagnostic archive is `/tmp/hexbane-ios-unsigned.xcarchive`.

## Source of truth in code

- `client:deploy-ios-simulator.sh` — simulator build, cached engine template, install and launch
- `client:export_presets.cfg` — iOS filters, output, bundle identifier and signing options
- `client:hexbane.csproj` — Godot .NET SDK and target framework
- `client:addons/rider-plugin/bin/rider-plugin.gdextension` — supported native platforms
