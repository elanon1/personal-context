---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, client, android, deploy]
sources: ["client:CLAUDE.md", "client:AGENTS.md", "client:deploy.sh", "client:export_presets.cfg"]
---

# Deploying the client to an Android phone

Server-side infra (docker, Nakama ports) is in [[infra-and-deploy]].

## Quick deploy

```bash
./deploy.sh
```
`deploy.sh` (verified 2026-09-07):
1. Uses `GODOT` (default `/Applications/Godot_mono47.app/Contents/MacOS/Godot`) and `ADB` (default `~/Library/Android/sdk/platform-tools/adb`); both must be executable (lines 7-18).
2. Refuses to build unless `adb devices` shows an authorized `device` (lines 20-25).
3. Reads `network/local_host` from the `[hexbane]` section of `project.godot` and warns when `http://$HOST:7350/` does not answer (lines 29-34).
4. Exports preset **Android** in debug, headless, to `hexbane1.apk` in the repo root, then `adb install -r` (lines 36-48).

Allow ~10 minutes; the APK is large (mostly `Resources/`).

## Server host on the device

`res://.env` is never packed (Godot ignores dotfiles), so on the phone every `NAKAMA_*` value falls back and `127.0.0.1` is unreachable. The host therefore comes from `project.godot`:
```
[hexbane]
network/local_host="192.168.1.34"
```
(`project.godot:68`, read by `NakamaClientManager.LocalHost()` at `NakamaClientManager.cs:154-155`). Set it to this machine's LAN IP (`ipconfig getifaddr en0`) and rebuild when it changes. In the editor `.env` `NAKAMA_HOST` still wins. Nakama must listen on `0.0.0.0:7349-7351`.

## Export presets (`export_presets.cfg`)

| Preset | Platform | Excludes | Output |
|---|---|---|---|
| Windows Desktop | Windows | SD race sheets + `frames_sd.tres` | `../export/test.exe` |
| Android | Android, package `pl.elanon.hexbane` (line 110) | HD race sheets + `frames.tres` (line 82) | `./hexbane1.apk` |
| macOS | macOS | SD set | `../export/test_mac.app` |
| iOS | iOS | HD set + Rider editor plugin | `../export/ios/hexbane.ipa` (see [[deploy-ios]]) |

Race sprite selection at runtime: [[vfx-and-race-animation]].

Enabled editor plugins (`project.godot:64`): `ColorPreview`, `godot_mcp`, `hexbane_android`. `addons/hexbane_android/export_plugin.gd` injects the `hexbane://` intent filter needed by Google sign-in ([[social-sign-in]]); it must stay enabled. `GodotPlayGameServices` is **disabled**; re-enabling it without a Play Console game id breaks the AAPT step (`string/game_services_project_id not found`). Toggle plugins from the editor, not by editing `project.godot` while the editor is open.

## Godot 4.7 and Google Play API 36 (2026-09-09)

Use `/Applications/Godot_mono47.app/Contents/MacOS/Godot` (4.7 stable .NET).
Main/test project SDK is `Godot.NET.Sdk/4.7.0`, targeting .NET 9. Matching Android,
iOS, Windows templates were installed; existing matching macOS template retained.
Android build template was replaced with stock 4.7, preserving the old tree at
`~/Library/Caches/hexbane/godot-4.7-upgrade/android-build-4.5.2`.
When replacing it manually, preserve/create `android/build/.gdignore` and executable
permission on `gradlew`; preferably use Godot's Install Android Build Template action.
Do not scan generated Android assets as Godot resources.

Stock 4.7 Android defaults: compileSdk **36**, targetSdk **36**, build tools **36.1.0**,
AGP 8.6.1, Gradle 8.11.1, Kotlin 2.1.21. Android SDK platform 36 installed during build.
Godot 4.7 editor Java SDK setting is
`/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`;
Android SDK is `~/Library/Android/sdk`.

Both Android presets explicitly target API 36 and exclude the desktop Rider plugin.
`Android Play`: AAB, package `com.dev.hexbane`, version code **5**.
`Android`: development APK, package `pl.elanon.hexbane`.

```bash
mkdir -p ../export/android
/Applications/Godot_mono47.app/Contents/MacOS/Godot --headless --path . \
  --export-release "Android Play" ../export/android/hexbane-godot47-api36-v5.aab
```

Verified export exit 0, bundletool API36/version5, signature verification; the Godot
warning about target36 exceeding default35 is absent. Other general C# experimental
support or editor shutdown warnings can still occur. No Play upload was performed.
For next releases increment Version Code, export AAB with Export With Debug unchecked,
and retain the same package ID and signing key.

Prior version3 was rejected for API35; version4 was rebuilt with target36 on 4.5.2.
The 4.7 upgrade also updates the underlying template defaults to36.
Google Play policy: https://support.google.com/googleplay/android-developer/answer/11926878

## Things that must stay as they are

- `display/window/per_pixel_transparency/enabled=false` (`project.godot:55`) and no `rendering/viewport/transparent_background`: a transparent window makes every scene change flash the task underneath on Android.
- `application/config/quit_on_go_back=false` (`project.godot:19`) with `SceneManager._Notification` handling Back; otherwise Back kills the process.
- Google client id/secret must be in `project.godot` `[hexbane] auth/*`, not only `.env`, or the APK shows the email form only.

## Startup and USB debugging repair (2026-09-12)

The OnePlus CPH2653 reproduced a startup failure in `com.dev.hexbane` version 10:
`System.ArgumentException: Unmanaged callbacks size mismatch`, followed by
`.NET: GodotPlugins initialization failed`. The native runtime was Godot 4.7 while
`hexbane.csproj` still selected `Godot.NET.Sdk/4.5.2`. The main project now selects
`Godot.NET.Sdk/4.7.0`, matching the installed editor/templates and the auth test project.
Always check the actual csproj and device logs; the earlier migration note was not
sufficient evidence of the current project setting.

The separate editor error `Could not create child process: scrcpy` came from enabled
Android mirroring with no scrcpy executable installed. On this Mac, scrcpy 4.1 was
installed with Homebrew and the running editor's `export/android/scrcpy/path` was set
to `/opt/homebrew/bin/scrcpy`. A child process launched by that editor connected to the
OnePlus over USB and exited successfully. This is an editor setting, not an APK dependency.

Validation: `dotnet build hexbane.csproj` completed with zero errors and 11 warnings;
Android debug export and Android Play release AAB export completed with exit 0.
The development APK was installed on the physical OnePlus and passed .NET, DI and
SceneManager startup, remaining alive beyond the former failure point. The normal
Android export still logs the missing optional `res://.env`; it did not cause this failure.
The export also reports existing editor-shutdown warnings after producing the artifact.

Release artifact: `../export/android/hexbane-v10-fixed.aab`. Version code remains 10;
if 10 has already been uploaded to Google Play, increment the code and export again
before uploading. A universal APK derived from this AAB is signed with the local debug
key for device validation; it is not signed by Google Play. No Play upload is performed.

## Troubleshooting

- `adb devices` says `unauthorized`: accept the USB-debugging prompt on the phone, or revoke authorizations in Developer options and re-plug.
- The export prints some non-critical `.tscn` parse errors; ignore them.
- Verify the intent filter landed: `aapt2 dump xmltree --file AndroidManifest.xml hexbane1.apk | grep -i scheme` or `adb shell pm resolve-activity --brief -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d hexbane://signed-in`.

## Source of truth in code
- `client:deploy.sh` — build + install script
- `client:export_presets.cfg` — presets, package name, per-platform sprite exclusions
- `client:project.godot` — `[hexbane]` settings, `quit_on_go_back`, transparency, enabled plugins
- `client:Application/Nakama/NakamaClientManager.cs` — `LocalHost()` fallback
- `client:addons/hexbane_android/export_plugin.gd` — Android manifest intent filter
