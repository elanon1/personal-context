---
type: project
project: Hexbane
area: client
status: guide
created: 2026-09-09
updated: 2026-09-16
verified: 2026-09-16
tags: [hexbane, android, google-play, signing, auth]
sources: ["client:export_presets.cfg", "client:deploy.sh", "client:Application/Authentication/AuthConfig.cs"]
---

# First Google Play internal release and Play Games configuration

This is guidance, not a record of completed console setup. No keys were generated, no application was created or uploaded. Existing persistent Nakama login works independently of Play Games. Native PGS remains disabled; integration and identity handling must be implemented and tested separately. Current Google guidance separates PGS platform identity from the primary in-game account; retain Nakama/Google Sign-In identity for progress.

## 1. Account and app

Register at https://play.google.com/console (currently USD 25 once), choose the account type matching the actual developer, and complete identity/device verification requested by Google. Create app: Hexbane, Game, desired language and pricing. The current Android Play preset uses com.dev.hexbane; the local Android preset uses pl.elanon.hexbane. Verify the package against the existing Play Console app before creating credentials; the first uploaded artifact fixes it for that Play app.

## 2. Upload key on this Mac

Create a private directory outside the repo, then generate an upload keystore:

```sh
mkdir -p "$HOME/.android/hexbane"
chmod 700 "$HOME/.android/hexbane"
keytool -genkeypair -v -storetype JKS \
  -keystore "$HOME/.android/hexbane/upload.jks" \
  -alias hexbane-upload -keyalg RSA -keysize 2048 -validity 10000
```

Answer the identity prompts; enter a strong password interactively and use the same password for the key and keystore (Godot requirement). Save the password in a password manager and keep a secure backup of the keystore. Do not commit passwords or private keys. If keytool is unavailable, configure the installed JDK; Godot 4.5 recommends OpenJDK 17.

Play App Signing should generate/manage the distribution signing key. The upload key authenticates uploads; the distribution key signs installed Play builds.

## 3. Godot export

Use the existing Android Play preset (com.dev.hexbane), after matching it to the Play Console app. Keep Gradle enabled. Choose AAB export format, set version name (e.g. 0.1.0) and increase version code for every uploaded build (current code is 2; next may be 3). Set Keystore Release to the absolute upload.jks path, Release User to hexbane-upload, and Release Password to the key password. Export release with Export With Debug unchecked, e.g. ../export/android/hexbane.aab. Existing deploy.sh generates a debug APK for USB, not the Play bundle.

## 4. Internal distribution and signing fingerprint

Play Console: Test and release > Testing > Internal testing > Create release. Enable Play App Signing using a Google-generated key, upload the signed AAB, resolve bundle validation errors and roll out to internal testers. Add your Google account to the tester list; open the opt-in URL using that account and install from Play. Internal testing can start before the full store listing is complete. New personal accounts need a separate qualifying closed test (currently 12 opted-in testers continuously for 14 days) before applying for production access; internal tests do not satisfy that requirement.

Find App integrity / App signing and copy SHA-1 from App signing key certificate, not Upload key certificate. A locally installed APK uses its own signing certificate, requiring another Android OAuth credential if testing that path. A differently signed existing local build may need uninstalling before the Play build installs; uninstall removes its local session.

## 5. Play Games

Grow users > Play Games Services > Setup and management > Configuration. Select the existing Google Cloud project used by Hexbane when applicable; configure consent and enable Google Play Games Services API. Add Android credential: the exact Play package (current Android Play preset: com.dev.hexbane) and the Play app-signing SHA-1. For the local pl.elanon.hexbane preset, create a separate credential with its actual signing certificate if needed. Create/select its Android OAuth client and save the association in Play Console. Copy the numeric Games Project ID displayed under the game name.

For server-side PGS access add Game server credential backed by a Web application OAuth client. Keep its client ID and downloaded client-secret JSON; the secret belongs only on the backend. Authorization codes are generated at runtime, never copied from the console. Add your device account under PGS Testers (separate from internal-distribution testers).

Later integration requires game ID in godot_play_game_services/game_id, server client ID in hexbane/auth/google_server_client_id, enabling/wiring the Android plugin and configuring server credentials. Do not simply replace existing Google identities with PGS identities; account continuity needs verification.

The first uploaded build can establish app signing before PGS is enabled. Upload a subsequent build with integration enabled and a higher version code for PGS testing. External testers need a reachable game server; the local LAN default in Hexbane is insufficient outside the LAN.

## Research verification (2026-09-16)

Read-only review of current code and official Google documentation; the actual Play Console account and deployed backend secrets were not inspected. ServiceBootstrapper still selects GoogleOAuthSignIn; project.godot does not enable GodotPlayGameServices. The bundled export plugin declares play-services-games-v2:21.0.0. PlayGamesSignIn is inactive and must not simply replace the primary account: current Google platform-authentication guidance explicitly separates PGS identity from the in-game account. Keep Nakama progress attached to the primary account and implement platform features separately; verify account continuity before rollout. Console setup alone does not activate the integration.

Enable PGS test access under Setup and management > Testers, either per account (including the developer) or by explicitly adding a release track. Distribution-track membership alone is not the PGS access configuration. Publish PGS separately from the application, allowing up to two hours for propagation.

## Official references

- https://support.google.com/googleplay/android-developer/answer/6112435
- https://support.google.com/googleplay/android-developer/answer/9859152
- https://support.google.com/googleplay/android-developer/answer/9845334
- https://support.google.com/googleplay/android-developer/answer/14151465
- https://developer.android.com/studio/publish/app-signing
- https://docs.godotengine.org/en/4.5/tutorials/export/exporting_for_android.html
- https://developer.android.com/games/pgs/console/setup
- https://developer.android.com/games/pgs/android/server-access
- https://developer.android.com/games/pgs/platform-authentication

## Source of truth in code

- client:export_presets.cfg — Android package, Gradle, version and empty PGS game ID
- client:deploy.sh — existing debug APK workflow
- client:Application/Authentication/AuthConfig.cs — OAuth configuration names
- client:Game/DI/ServiceBootstrapper.cs — current Google browser flow
- client:addons/GodotPlayGameServices/export_plugin.gd — game ID and manifest injection
