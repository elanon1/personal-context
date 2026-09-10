---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, client, auth, google, android]
sources: ["client:docs/client/social-sign-in.md", "client:CLAUDE.md"]
---

# Google sign-in (client)

Server contract: hexbane-server `docs/client/google-auth.md` (see [[rpcs]] for the account RPCs). The client uses Nakama's **built-in** `AuthenticateGoogleAsync` (`GoogleAuthGateway.cs:36`); there is no custom auth RPC. Email/password stays as the fallback (`LoginService.cs:68`, `AuthenticateEmailAsync`) and is what the seeded per-race test accounts use.

## Pieces

| Piece | File |
|---|---|
| Browser flow (RFC 8252 loopback + PKCE) | `Application/Authentication/Social/GoogleOAuthSignIn.cs` |
| Provider selection | `Game/DI/ServiceBootstrapper.cs:47-59` (`GoogleOAuthSignIn` if `AuthConfig.HasGoogleClient`, else `NullSocialSignIn`) |
| Token → Nakama session | `Application/Authentication/Gateways/GoogleAuthGateway.cs` |
| Session cache and refresh | `Core/Auth/SessionStore.cs`, `LoginService.TryRestoreSession` (`LoginService.cs:188-208`), `LoginService.RefreshSession` (`:253-269`) |
| Config | `Application/Authentication/AuthConfig.cs` |
| Screen | `Game/ScenesV3/Auth/LoginPanel.cs` (+ `RegisterPanel`) |
| Android manifest | `addons/hexbane_android/export_plugin.gd` |

Play Games is not used. `PlayGamesSignIn.cs`, `icon_play_games.png` and the disabled `addons/GodotPlayGameServices` remain only so it can be switched back on (`AuthConfig.GoogleServerClientId`, `AuthConfig.cs:62-66`, is the setting it would need).

## Configuration

| Key | Sources (env first, then project setting) |
|---|---|
| `GOOGLE_CLIENT_ID` | `.env` or `hexbane/auth/google_client_id` (`AuthConfig.cs:29`) |
| `GOOGLE_CLIENT_SECRET` | `.env` or `hexbane/auth/google_client_secret` (`:36`) |
| `GOOGLE_LOOPBACK_PORT` | optional pin, `.env` or `hexbane/auth/google_loopback_port` (`:51`) |

Google Cloud: create a **Desktop app** OAuth client; consent screen scopes `openid email profile`; while in Testing every player must be a listed test user. No redirect URI to register (Google waives the port for loopback on this client type). The Desktop client secret ships on purpose (Google distributes it in `client_secret.json`, PKCE secures the exchange). The server's **Web application** secret never enters this repo. This works because Nakama 3.27 does not validate the ID token `aud` claim (`social/social.go`).

Both settings are present in `project.godot:70-72` (needed because `.env` is not exported). With nothing configured the screen shows only the email form.

## Flow

1. `OS.ShellOpen` the Google auth URL; a `TcpListener` on `127.0.0.1:<port>` waits for the redirect (`GoogleOAuthSignIn.cs:106-111`). `HttpListener` is avoided (unreliable on Android .NET, needs URL reservations on Windows).
2. The redirect is captured on the thread pool (`Task.Run`, `ConfigureAwait(false)`, lines 135-167), because on Android the Godot main loop stops iterating while Chrome is in front.
3. On Android the redirect is answered with `302` to `intent://signed-in#Intent;scheme=hexbane;package=<running-application-id>;S.browser_fallback_url=http://127.0.0.1:<port>/done;end` (lines 210-212); the fallback page `/done` offers a "Return to the game" button. The scheme resolves only because the export plugin adds the `VIEW/BROWSABLE` intent filter (`export_plugin.gd:35-46`); hand-edits to `android/build/AndroidManifest.xml` are dropped by the manifest merger.
4. The code exchange runs only once the game is in front again (OEMs cut DNS for backgrounded apps). Connection-level failures are retried with a 2 s delay (`ExchangeRetryDelay`, line 73); anything that may have reached Google is not retried.
5. `GoogleAuthGateway` calls `AuthenticateGoogleAsync(idToken)`; the session (with refresh token) is cached in `SessionStore`, tagged with the issuing server. Email sessions are cached as well, including test-account sign-ins. Use Logout to switch accounts. Passwords and Google credentials are not stored in this cache.

Startup order: select the cached session's server if supported by the selector (unless dev auto-login is enabled) → cached session (refreshed via `SessionRefreshAsync` within five minutes of access-token expiry) → silent platform sign-in (nothing implements it today) → show the button. Controls cannot switch server or start a test-account login during restoration.

The cache is `user://auth_session.cfg` and contains access/refresh tokens, provider and server name. Existing Google cache files remain compatible. A refresh rejection with HTTP 401/403 or a fully expired session removes the cache; network errors and server outages retain it for a later launch. Explicit Logout removes it. Session-health failure returns through the login flow while retaining renewable credentials. Successful SDK-triggered token refreshes are also persisted. A failed socket connection returns a failed login result and clears in-memory authentication while retaining the saved tokens.

Related Android settings: `application/config/quit_on_go_back=false` plus `SceneManager._Notification` (Back navigates instead of quitting and exposing the leftover Chrome tab). The Chrome tab stays in the app switcher after sign-in; harmless.

## Brand asset

`Resources/Images/Auth/icon_google.png` is Google's `googleg_standard_color_128dp` (2x), stored unmodified as their terms require; the white disc is a separate node built in `LoginPanel.ApplyProviderMark`.

## Android return package (2026-09-09)

`AuthConfig.AndroidPackage` reads `Engine.GetSingleton("AndroidRuntime").getApplicationContext().getPackageName()` on the main thread. Both the automatic intent redirect and fallback button target that application id. Local preset uses `pl.elanon.hexbane`, Android Play uses `com.dev.hexbane`; previously the local package was hardcoded and Play's return intent targeted the wrong app. The `hexbane` scheme remains declared by the Android export plugin. Missing runtime/package now fails sign-in explicitly and closes the loopback listener instead of opening an invalid return link.

Regression verification: mocked AndroidRuntime bridge with each package, checked both generated intent links. Before fix both Play checks failed; after fix all21 auth checks pass. Main/Auth compilation: zero errors,11 existing warnings. No ADB device connected; real Chrome/Google return and Play-installed build not exercised. Re-export and upload a higher versionCode to distribute this client fix.

## Production WebSocket TLS (2026-09-09)

`NakamaClientManager.SetupSocket` explicitly uses `Socket.From(client, new WebSocketStdlibAdapter())`. NakamaClient 3.16.0's parameterless-adapter factory still selects the legacy `WebSocketAdapter`; against production it rejected the certificate with `RemoteCertificateValidationCallback`, although HTTPS email authentication succeeded. Native .NET WebSocket connected successfully with standard certificate validation. Do not disable TLS verification to work around this error.

Verified old/new adapters against the same production session (existing test account, create=false), then real Godot4.7 LoginPanel → LoginService → NakamaClientManager production login: socket connected and login succeeded. Main/Auth build: zero errors, 11 existing warnings; all 17 auth regression checks pass. Existing exported applications need rebuilding to receive the fix; physical-device and full duel checks were not performed.

## Known gaps

- Mid-session expiry returns through the login flow to refresh and reconnect; seamless recovery inside an active duel is not implemented.
- One account per provider: Nakama matches on provider id, never email, so Google and email logins are different accounts. Account linking is a server plan.
- No silent sign-in exists.

## Verification (2026-09-09)

`dotnet build Tests/Auth/Auth.csproj`, then Godot 4.5.2 .NET headless with `--path Tests/Auth` runs regression checks against real session persistence and LoginService, with a controlled HTTP adapter for failures. `-- write` then `-- read` verifies persistence across processes. `-- live-write` then `-- live-read` uses local Nakama at 127.0.0.1:7350, creates an isolated test email account, restores it in another process, refreshes using its real refresh token, and verifies revocation. The test project uses a separate Godot user-data directory.

All checks passed; main build has 0 errors and 9 existing warnings. The service test harness emits a Godot ObjectDB cleanup warning at exit (verbose output identifies the LoginService signal object). Full browser Google OAuth and physical Android restart were not exercised in this task.

## Runtime connection recovery (HEX-6, 2026-09-10)

`NakamaClientManager.EnsureConnected` serializes recovery, refreshes an access token expiring within one minute, and connects a replacement socket when disconnected. The SDK updates the session in place; LoginService's existing `ReceivedSessionUpdated` subscription persists token rotations. Recovery checks that both the session object and selected client still match before publishing the socket, preventing a pending connection from restoring a logged-out session. Connection establishment has a ten-second timeout; closing the previous transport is bounded to three seconds.

`GameContext` uses recovery in the existing 30-second health check. Mobile `NotificationApplicationResumed` forces transport replacement, including connections that still appear open after suspension. Temporary network failures preserve the character and cached session. A missing/nonrenewable session or rejected refresh returns through the login flow; it does not silently delete cached credentials on a transient outage.

Both AI and PvP matchmaking await recovery before using the current socket. ModeOverlay awaits and handles creation failures for AI, PvP and requeue, returning to mode selection with an error dialog. Cancel is disabled while initial creation is pending. Replacing a socket emits the close/established lifecycle so notification subscriptions rebind. Active PvP queue intent is restored on the replacement socket; cancel/logout invalidates pending requests, late tickets are removed, and obsolete failures do not overwrite the current UI. A different user's session cannot inherit queue intent.

Verification: `Game/ScenesV3/Dev/ConnectionRecoveryVerification.tscn` runs against local seeded Nakama with `HEXBANE_IGNORE_ENV_FILE=1 DEV_AUTO_LOGIN=false NAKAMA_HOST=127.0.0.1`. It covers closed-socket AI/PvP, expired-token renewal, concurrent recovery, synthetic app resume, queued resume/cancellation, notification socket rebinding, cancellation during queue creation, logout invalidation and error-overlay recovery. The original AI reproduction failed with `Socket is not connected` before the fix. Existing Auth tests cover persistence and refresh/cache failure policies. Physical Android suspension and notification delivery remain manual follow-ups; the headless run reports an audio playback/resource warning for `Resources/Music/menu/game_found.wav` at exit. Build: zero errors and 11 existing warnings. Live regression: 10 checks pass; existing Auth verifier: 21 checks pass. Logs are in client `verification/connection-recovery/`.

Source files: `client:Application/Nakama/NakamaClientManager.cs`, `client:Game/Autoloads/GameContext.cs`, `client:Application/ArcaneDuel/{Bot/BotMatchManager,Normal/MatchManager}.cs`, `client:Game/ScenesV3/Dashboard/ModeOverlay.cs`, `client:Game/ScenesV3/Dev/ConnectionRecoveryVerification.cs`.

## Source of truth in code
- `client:Application/Authentication/Social/GoogleOAuthSignIn.cs` — loopback flow, Android intent redirect, retry policy
- `client:Application/Authentication/Gateways/GoogleAuthGateway.cs` — `AuthenticateGoogleAsync`
- `client:Application/Authentication/AuthConfig.cs` — keys, `AndroidPackage`, `AndroidScheme`
- `client:Application/Authentication/LoginService.cs`, `Core/Auth/SessionStore.cs` — restore/refresh
- `client:Game/DI/ServiceBootstrapper.cs` — provider choice
- `client:addons/hexbane_android/export_plugin.gd` — manifest intent filter
- `client:project.godot` — `[hexbane] auth/*`, `quit_on_go_back`

- `client:Tests/Auth/` — isolated Godot regression and local Nakama integration checks
