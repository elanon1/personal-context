---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-09
verified: 2026-09-09
tags: [hexbane, server, auth, google, nakama]
sources: ["server:docs/client/google-auth.md", "server:docs/superpowers/specs/2026-09-04-google-auth-design.md", "client:CLAUDE.md"]
---

# Authentication contract

Nakama's **built-in** authenticate endpoints do all the work. The plugin registers no auth RPC; it only
adds logging hooks. What is implemented today:

| Provider | Client call | Server side | Status |
|---|---|---|---|
| Email / password | `AuthenticateEmailAsync(email, pw, null, create)` (`client:Application/Authentication/LoginService.cs:68`, `RegisterService.cs:51`) | `Before/AfterAuthenticateEmail` log only (`server:modules/auth/email_hooks.go`) | live |
| Device id | `AuthenticateDeviceAsync(OS.GetUniqueId(), null, true)` (`LoginService.cs:114`) | none | dev / test harness only (`TutorialVerification`) |
| Google account (ID token) | `AuthenticateGoogleAsync(idToken)` (`client:Application/Authentication/Gateways/GoogleAuthGateway.cs:36`) | `Before/AfterAuthenticateGoogle` log the credential kind (`server:modules/auth/social_hooks.go:47-59`) | live on desktop and Android |
| Play Games (auth code) | `PlayGamesSignIn.cs` kept but never constructed; addon disabled | same Google hook; needs `GOOGLE_CREDENTIALS_JSON` | **not used** |
| Sign in with Apple | none | `Before/AfterAuthenticateApple` (log), needs `APPLE_BUNDLE_ID` | server hooks only, no client |
| Game Center | none | `Before/AfterAuthenticateGameCenter` (log) | server hooks only, no client |

## Google sign-in as implemented

Client (`client:Application/Authentication/Social/GoogleOAuthSignIn.cs`):
1. RFC 8252 loopback flow with PKCE: open the system browser, listen on `127.0.0.1:<free port>` over a
   raw `TcpListener`, exchange the code at `https://oauth2.googleapis.com/token`, scope `openid email profile`.
2. The OAuth client is a **Desktop app** client. Its id and secret ship in `.env` (`GOOGLE_CLIENT_ID`,
   `GOOGLE_CLIENT_SECRET`) and in `project.godot` `[hexbane] auth/google_client_id|secret` (only the project
   settings survive export). The **web** client secret is a different value and stays on the server as
   `GOOGLE_CREDENTIALS_JSON`. The old client contract's "no secret in the client / use the web client id"
   rule is therefore superseded — see `client:CLAUDE.md` "Google sign-in".
3. The resulting **ID token** goes to `AuthenticateGoogleAsync`. Nakama v3.27.0 does not validate the
   token's `aud`, so a token minted for the desktop client is accepted (claim from `client:CLAUDE.md`,
   not re-verified here).
4. Android: redirect answered with `302` to `intent://signed-in#Intent;scheme=hexbane;…`; the `hexbane`
   scheme is declared by the `addons/hexbane_android` export plugin; the flow runs off Godot's
   synchronization context and the token exchange waits until the game is in front again.
5. With no Google client id configured, `NullSocialSignIn` hides the button and only the email form shows.

Server hook (`server:modules/auth/social_hooks.go:37-45`): `credentialKind` classifies a 3-segment value as
`id_token`, anything else as `play_games_auth_code`, and logs it; `After…` logs `out.Created`.

## Sessions

- Nakama runs with `--session.token_expiry_sec 7200` (`server:docker-compose.yml`), refresh tokens on
  every provider.
- Client caches email and social sessions (`SessionStore`) with provider and server name; `TryRestoreSession`
  (`LoginService.cs:188-221`) ignores a cache from another server, refreshes via `SessionRefreshAsync`
  when within 5 minutes of expiry, and falls back to the sign-in screen on failure. Transient failures retain the cache; HTTP 401/403 refresh rejection discards it. Explicit logout clears the cache. See [[social-sign-in]] for restart verification against local Nakama.
- After every sign-in: `SetupSocket` → `socket.ConnectAsync(session, appearOnline: true, 60)`
  (`client:Application/Nakama/NakamaClientManager.cs:53-70`).

## One human, several accounts (accepted)

Each provider stores its identity in its own column (`users.email`, `users.google_id`, …) and Nakama
never matches on email, so switching sign-in method creates a new account with a new character.
Account linking by email is out of scope and unimplemented.

## Server configuration (`server:docker-compose.yml:20-44`)

- `GOOGLE_CREDENTIALS_JSON` → `--google_auth.credentials_json` (enables Play Games auth-code exchange only; not needed for ID tokens).
- `APPLE_BUNDLE_ID` → `--social.apple.bundle_id`.
- Both optional; entrypoint runs `sh -ec` (no `-x`) so the secret is not traced into logs.

## After the session exists

`get_my_character` decides the route: `success:true` → dashboard; `"No character found for user"` →
character creation (see [[race-selection]]).

## Source of truth in code
- `server:modules/auth/init.go`, `email_hooks.go`, `social_hooks.go` — registered hooks (logging only).
- `server:docker-compose.yml` — Nakama flags and session expiry.
- `client:Application/Authentication/LoginService.cs`, `RegisterService.cs` — email/device/social entry points, session restore.
- `client:Application/Authentication/Social/GoogleOAuthSignIn.cs`, `NullSocialSignIn.cs`, `PlayGamesSignIn.cs` — the loopback flow and the disabled Play Games path.
- `client:Application/Authentication/Gateways/GoogleAuthGateway.cs` — `AuthenticateGoogleAsync`.
- `client:Application/Authentication/AuthConfig.cs` — where ids/secrets are read from.
- `client:docs/client/social-sign-in.md`, `client:CLAUDE.md` — Android caveats.
