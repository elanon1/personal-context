---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-16
updated: 2026-09-16
verified: 2026-09-16
tags: [hexbane, client, ios, auth, game-center, apple]
sources: ["client:export_presets.cfg", "client:hexbane.csproj", "server:modules/auth"]
---

# iOS Game Center and Sign in with Apple

This note is the setup runbook for the future iOS identity flow. The intended order is:

1. Try Game Center silently when the app starts.
2. If Game Center is unavailable or unauthenticated, offer Sign in with Apple.
3. Keep Google browser sign-in as an optional additional fallback.
4. Send the resulting credential to Nakama and restore the same cached session on later launches.

Game Center and Sign in with Apple are different identities. They must not be merged by email or by
display name. If one Hexbane account should support both, add an explicit, authenticated account-link
flow later.

## Game Center setup

### Apple Developer

1. Open [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/identifiers/list).
2. Select the App ID whose bundle identifier is the iOS app bundle identifier (`com.dev.hexbane` in the current preset).
3. Enable the **Game Center** capability and save.
4. Regenerate or refresh the development/distribution provisioning profile after changing capabilities.

Game Center authentication does not use a Google OAuth client, a `.p8` Apple Sign in key, or the
Play Games SHA-1. The iOS signing team and the bundle identifier must match the installed build.

### App Store Connect

1. Open the Hexbane app in [App Store Connect](https://appstoreconnect.apple.com/).
2. Open **Features → Game Center** and create or select the Game Center group for the app.
3. Add leaderboards and achievements only when the corresponding feature is implemented in the client.
4. For development, use a sandbox Game Center account and the same Apple team that signs the build.

### Xcode/Godot export

The exported Xcode project must contain the Game Center capability/entitlement. In a signed device
build, open `hexbane.xcodeproj`, select the app target, add **Game Center** under **Signing &
Capabilities**, select the correct team, and let Xcode refresh the profile. Godot's current iOS
preset/export does not yet add a Game Center client bridge.

The native client work still required is:

- call `GKLocalPlayer.authenticateHandler` at startup;
- handle the returned view controller on the main thread when Apple asks for user interaction;
- after authentication, obtain the Game Center player identity and the authentication payload required by Nakama;
- call Nakama's `AuthenticateGameCenterAsync` with the player ID, bundle ID, timestamp, salt, and signature;
- cache the Nakama session, not the Game Center private material.

Do not treat `GKLocalPlayer.isAuthenticated` as proof of a Nakama session. It only proves the Apple
platform identity; the server authentication call still has to succeed.

## Sign in with Apple setup

### Apple Developer

1. In **Certificates, Identifiers & Profiles → Identifiers**, open the Hexbane App ID and enable **Sign in with Apple**.
2. Create a **Sign in with Apple key**. Record the **Key ID** and **Team ID**; download the `.p8` file once and store it as a server secret.
3. If a web or browser callback is needed, create a **Services ID**, enable Sign in with Apple for it, and register the exact HTTPS return URL. A native iOS-only flow can use the App ID without a Services ID.
4. Refresh the provisioning profile after enabling the capability.

Never put the `.p8` key, Apple client secret JWT, or private key in the Godot project, AAB/IPA assets,
Git, or chat. The backend should generate short-lived client-secret JWTs from the `.p8` key and
validate Apple's identity token.

### Client flow

The client needs a native iOS bridge for `ASAuthorizationAppleIDProvider`:

1. Request `ASAuthorizationAppleIDRequest` with the required scopes.
2. Present the authorization controller from the active iOS view controller.
3. Receive the identity token and authorization code in the completion callback.
4. Pass the identity token to Nakama's Apple authentication endpoint.
5. Handle cancellation separately from a configuration or network failure.

The current Hexbane client has no Apple provider or native bridge. The server has Apple auth hooks,
but hooks only log the request; they do not create a client credential or replace the built-in Nakama
authentication endpoint.

### Server configuration

Set the bundle identifier in the Nakama deployment:

```text
APPLE_BUNDLE_ID=com.dev.hexbane
```

The local Docker Compose deployment supports this through the `APPLE_BUNDLE_ID` environment variable
and the `--social.apple.bundle_id` startup flag. The current production Helm template does not yet
pass this flag, so it must be added when the Apple client is implemented. The Apple `.p8` key is not
currently required by Nakama's built-in Apple identity-token endpoint. If a future flow exchanges authorization codes or generates
Apple client-secret JWTs on the backend, add those values as a separate Kubernetes Secret.

## Hexbane fallback flow

The release login screen should expose only platform and SSO controls:

```text
Connecting to server…
Checking Game Center / Play Games…
Authenticating…
Loading your account…
```

Android tries Play Games first, then offers Google. iOS should currently skip Game Center and offer
Google until the native Game Center and Apple bridges are implemented. Once those bridges exist, iOS
should try Game Center first and offer Apple as the primary fallback. A failed provider must leave the
retry button available and must not delete a valid cached Nakama session.

## Testing checklist

- Use a signed iOS build with the correct team, bundle ID, and refreshed provisioning profile.
- Test on a real iPhone; the current unsigned simulator workflow does not prove Game Center or Apple credentials.
- Use the Apple sandbox/test account and the same Game Center-enabled app record.
- Test first launch, returning launch, cancellation, no-network, revoked session, and a second device.
- Check Nakama logs for the provider and verify that the session reaches `get_my_character`.
- Confirm that switching between Game Center, Apple, and Google does not silently replace an existing account.

## Current Hexbane status

- Android Play Games server authentication is live and uses `GOOGLE_CREDENTIALS_JSON` in Kubernetes.
- iOS Game Center client integration: implemented in `GameCenterSignIn` with a native GameKit
  framework bridge; the provider attempts silent authentication at startup and falls back after
  timeout/error.
- iOS Sign in with Apple client integration: not implemented; server hooks exist.
- iOS current fallback: Google browser flow after Game Center failure.
- Account linking between providers: not implemented.

### Game Center bridge implementation (2026-09-16)

- `addons/GodotPlayGameServices/export_plugin.gd` embeds the GameKit bridge framework and adds the
  iOS Game Center export dependency.
- `Application/Authentication/Social/GameCenterSignIn.cs` polls the native authentication state,
  obtains the identity verification signature, and passes the complete credential to Nakama.
- `GoogleAuthGateway` calls Nakama `AuthenticateGameCenterAsync` for this provider; Google and Play
  Games continue using their existing endpoint.
- The simulator build was ad-hoc signed with the Game Center entitlement and launched. GameKit
  attempted authentication and returned the simulator's server/account errors; after 30 seconds
  the login screen returned to the Game Center button. A real signed device or Apple sandbox
  account is still required for successful credential verification.

### Verification on 2026-09-16

- App Store Connect app `HexbaneDev` (iOS app record 6812841408) shows the Game Center group
  attached, with the app listed under Test Attached Apps. No leaderboards or achievements were
  created because the client does not implement those features yet.
- The simulator workflow exported and built the iOS Xcode project successfully, installed
  `com.dev.hexbane` on iPhone 17 Pro (iOS 26.5), and launched it. This was a debug simulator build;
  it still displays the debug test accounts and Local server selector.
- The running simulator build did not open Game Center. This is expected until the native
  `GKLocalPlayer.authenticateHandler` bridge and signed entitlements are added; App Store Connect
  configuration alone cannot initiate platform login.

## Source of truth in code

- `client:Game/DI/ServiceBootstrapper.cs` — provider selection
- `client:Application/Authentication/Social/ISocialSignIn.cs` — provider interface
- `client:Application/Authentication/Social/PlayGamesSignIn.cs` — Android PGS provider
- `client:Application/Authentication/Social/GoogleOAuthSignIn.cs` — browser fallback
- `client:Application/Authentication/Gateways/GoogleAuthGateway.cs` — Google/PGS Nakama gateway
- `client:Game/ScenesV3/Auth/LoginPanel.cs` — status and retry UI
- `client:export_presets.cfg` — iOS bundle/signing settings
- `server:modules/auth/social_hooks.go` — Apple and Game Center logging hooks
