---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-08
updated: 2026-09-09
verified: 2026-09-09
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
simulator (preferring an already booted one), and launches `com.hexbane.game`.
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

## Development signing (physical devices only)

Turn off **Export Project Only** and restore Debug Code Sign Identity to blank /
`Apple Development` when a signed device IPA is wanted.

The bundle identifier is `com.hexbane.game`. Xcode needs an Apple Development certificate,
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
