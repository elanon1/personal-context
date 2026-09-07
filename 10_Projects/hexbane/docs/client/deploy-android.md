---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
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
1. Uses `GODOT` (default `/Applications/Godot.app/Contents/MacOS/Godot`) and `ADB` (default `~/Library/Android/sdk/platform-tools/adb`); both must be executable (lines 7-18).
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
| iOS | iOS | HD set | (none) |

Race sprite selection at runtime: [[vfx-and-race-animation]].

Enabled editor plugins (`project.godot:64`): `ColorPreview`, `godot_mcp`, `hexbane_android`. `addons/hexbane_android/export_plugin.gd` injects the `hexbane://` intent filter needed by Google sign-in ([[social-sign-in]]); it must stay enabled. `GodotPlayGameServices` is **disabled**; re-enabling it without a Play Console game id breaks the AAPT step (`string/game_services_project_id not found`). Toggle plugins from the editor, not by editing `project.godot` while the editor is open.

## Things that must stay as they are

- `display/window/per_pixel_transparency/enabled=false` (`project.godot:55`) and no `rendering/viewport/transparent_background`: a transparent window makes every scene change flash the task underneath on Android.
- `application/config/quit_on_go_back=false` (`project.godot:19`) with `SceneManager._Notification` handling Back; otherwise Back kills the process.
- Google client id/secret must be in `project.godot` `[hexbane] auth/*`, not only `.env`, or the APK shows the email form only.

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
