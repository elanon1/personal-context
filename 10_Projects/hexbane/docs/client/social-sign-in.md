---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
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
3. On Android the redirect is answered with `302` to `intent://signed-in#Intent;scheme=hexbane;package=pl.elanon.hexbane;S.browser_fallback_url=http://127.0.0.1:<port>/done;end` (lines 210-212); the fallback page `/done` offers a "Return to the game" button. The scheme resolves only because the export plugin adds the `VIEW/BROWSABLE` intent filter (`export_plugin.gd:35-46`); hand-edits to `android/build/AndroidManifest.xml` are dropped by the manifest merger.
4. The code exchange runs only once the game is in front again (OEMs cut DNS for backgrounded apps). Connection-level failures are retried with a 2 s delay (`ExchangeRetryDelay`, line 73); anything that may have reached Google is not retried.
5. `GoogleAuthGateway` calls `AuthenticateGoogleAsync(idToken)`; the session (with refresh token) is cached in `SessionStore`, tagged with the issuing server. Email sessions are deliberately not cached so the per-race test accounts can be switched.

Startup order: cached session (refreshed via `SessionRefreshAsync` if expired) → silent platform sign-in (nothing implements it today) → show the button.

Related Android settings: `application/config/quit_on_go_back=false` plus `SceneManager._Notification` (Back navigates instead of quitting and exposing the leftover Chrome tab). The Chrome tab stays in the app switcher after sign-in; harmless.

## Brand asset

`Resources/Images/Auth/icon_google.png` is Google's `googleg_standard_color_128dp` (2x), stored unmodified as their terms require; the white disc is a separate node built in `LoginPanel.ApplyProviderMark`.

## Known gaps

- Mid-session expiry is not handled: `GameContext.CheckSessionHealth` (`GameContext.cs:117-125`) logs the player out instead of refreshing, because the live match socket would need replacing.
- One account per provider: Nakama matches on provider id, never email, so Google and email logins are different accounts. Account linking is a server plan.
- No silent sign-in exists.

## Source of truth in code
- `client:Application/Authentication/Social/GoogleOAuthSignIn.cs` — loopback flow, Android intent redirect, retry policy
- `client:Application/Authentication/Gateways/GoogleAuthGateway.cs` — `AuthenticateGoogleAsync`
- `client:Application/Authentication/AuthConfig.cs` — keys, `AndroidPackage`, `AndroidScheme`
- `client:Application/Authentication/LoginService.cs`, `Core/Auth/SessionStore.cs` — restore/refresh
- `client:Game/DI/ServiceBootstrapper.cs` — provider choice
- `client:addons/hexbane_android/export_plugin.gd` — manifest intent filter
- `client:project.godot` — `[hexbane] auth/*`, `quit_on_go_back`
