---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-17
updated: 2026-09-17
verified: 2026-09-17
tags: [hexbane, client, ios, xcode, signing, apple-developer]
sources: ["client:export_presets.cfg", "godot:editor/export/editor_export_platform_apple_embedded.{cpp,h}"]
---

# Apple code signing and the Godot → Xcode pipeline

Companion to [[deploy-ios]] (commands) and [[ios-game-center-apple-auth]] (Game Center). This note is
the *model*: what each Apple object is, how the Godot preset turns into Xcode settings, and which
combination to use for each kind of build.

## The objects

Apple will only run code on an iPhone if it can prove **who built it**, **which app it is**,
**what it may do** and **where it may run**. Four objects carry that proof; a build fails when any
of them disagrees with the others.

| Object | Answers | Lives in | Hexbane today |
|---|---|---|---|
| **Signing identity** = certificate + private key | who built it | your Mac's keychain (the key never leaves it) and the Developer portal (the cert) | `Apple Development: filip.pokoj@gmail.com (TDGVSTK589)` — one valid identity; no *Apple Distribution* yet |
| **App ID** | which app, and its **capabilities** | Developer portal → Identifiers | `com.dev.hexbane` on team `4JC7VY2984`, Game Center enabled |
| **Entitlements** | what the binary claims it may do | `hexbane.entitlements` in the export, baked into the signature | `com.apple.developer.game-center = true` (Godot writes it from the preset's capabilities) |
| **Provisioning profile** | the contract: *this* App ID, signed by *these* certs, with *these* entitlements, on *these* devices | portal; embedded in the app as `embedded.mobileprovision` | `iOS Team Provisioning Profile: com.dev.hexbane`, auto-created, valid to 2027-09-17, covers the registered iPhone |

Rules that follow from the table:

- A **development** profile lists device UDIDs; the app installs only on those. An **App Store**
  profile lists none (TestFlight/App Store do the gating), and is signed with the *Distribution*
  certificate. **Ad hoc** = distribution cert + device list (rarely useful now that TestFlight exists).
- Every entitlement in the binary must be allowed by the profile, and the profile can only allow
  what the App ID has enabled. Adding a capability therefore means: enable it on the App ID →
  regenerate the profile → the entitlements file claims it. Automatic signing does all three when
  you tick the capability in Xcode; Godot's exporter only writes the entitlements file.
- The private key is the one thing you cannot re-download. If the Mac is lost, revoke the
  certificate and let Xcode create a new one; nothing else needs to change.
- **Automatic signing** (`CODE_SIGN_STYLE = Automatic` + `DEVELOPMENT_TEAM`) lets Xcode create and
  refresh certs and profiles through your Apple ID (`-allowProvisioningUpdates` on the command
  line, or the account in *Xcode → Settings → Accounts*). It requires the identity setting to be
  the *generic* `Apple Development`; a manually named identity such as `Apple Distribution` makes
  Xcode stop with *conflicting provisioning settings*. **Manual signing** means you pick the
  profile by name and own its lifecycle — CI use only.
- `-` as the identity is *ad-hoc signing*: a signature with no certificate. Simulators and macOS
  accept it; a physical iPhone never does. This is what the simulator script wants.

## How the Godot preset becomes Xcode settings

Godot copies a template `project.pbxproj` and substitutes placeholders
(`editor/export/editor_export_platform_apple_embedded.cpp`). What matters for signing:

| Preset option | Effect |
|---|---|
| `application/app_store_team_id` | `DEVELOPMENT_TEAM` in both configurations |
| `application/code_sign_identity_debug` / `_release` | `CODE_SIGN_IDENTITY` for the Debug / Release configuration. Empty → `Apple Development` / `Apple Distribution` |
| `application/provisioning_profile_specifier_*`, `_uuid_*` | `PROVISIONING_PROFILE_SPECIFIER` / `PROVISIONING_PROFILE` |
| (derived) | `CODE_SIGN_STYLE` = **Manual** when a profile is given *or* the identity is anything other than the two generic names (so `-` ⇒ Manual); otherwise **Automatic** |
| `application/export_method_debug` / `_release` | only `export_options.plist` (`app-store`, `development`, `ad-hoc`, `enterprise`) — used when Godot itself runs `xcodebuild -exportArchive`, i.e. **not** with Export Project Only |
| `application/export_project_only` | stop after generating `hexbane.xcodeproj`; no `xcodebuild`, no IPA |
| `capabilities/*` | entries in `hexbane.entitlements` and `Info.plist` |

Consequences for this project:

- The scheme Godot generates runs **Release** (Run, Profile, Archive). The Debug configuration is
  only reached by `xcodebuild -configuration Debug`.
- Debug identity `-` + Export Project Only is what `deploy-ios-simulator.sh` requires and checks.
  Leave it.
- Release identity **must be `Apple Development`** for a device Run with automatic signing. The
  default (empty → `Apple Distribution`) is why the first device build needed the signing panel
  changed by hand, and why that change vanished at the next export (Godot rewrites the pbxproj).
- Nothing in the pbxproj is worth editing: every export regenerates it. Put the setting in the
  preset, or override on the `xcodebuild` command line.
- Project settings vs `.env`: `.env` is never exported; anything the phone needs comes from
  `project.godot` (`[hexbane]` section) — see [[deploy-ios]].

## The three builds and their settings

| Build | Preset | Signing | Command / action |
|---|---|---|---|
| **Simulator** | debug identity `-`, Export Project Only | ad-hoc | `./deploy-ios-simulator.sh` |
| **Your iPhone (dev)** | release identity `Apple Development`, team id set, Export Project Only | Automatic, development profile, `get-task-allow` so Xcode can attach | `--export-release iOS ../export/ios/hexbane.xcodeproj`, then Xcode ▶ with the phone selected, or the `xcodebuild … -allowProvisioningUpdates build` + `devicectl` recipe in [[deploy-ios]] |
| **TestFlight / App Store** | same preset | Automatic; Xcode creates the *Apple Distribution* cert and App Store profile at first upload | Xcode *Product → Archive → Distribute App → App Store Connect*, or `xcodebuild archive` + `-exportArchive -exportOptionsPlist hexbane/export_options.plist` (already says `method = app-store`, team set) |

Before the first TestFlight upload: 1024×1024 icon in the preset (`icons/icon_1024x1024`, currently
empty — App Store Connect rejects the build without it), `application/short_version` and
`application/version` (currently empty → `1.0.0`/`1.0.0`; the build number must increase per upload),
and the App Store Connect record (`HexbaneDev`, app id 6812841408, bundle `com.dev.hexbane`).

## Apple Developer portal — what exists and what is still open

Done (verified from the profile embedded in the 2026-09-17 device build):

- Team `4JC7VY2984`, Apple Development certificate for filip.pokoj@gmail.com.
- App ID `com.dev.hexbane` with **Game Center**; team provisioning profile with that entitlement.
- Filip's iPhone 15 Pro Max registered (automatic, when Xcode first ran to it). Agata's iPhone gets
  registered the same way the first time it is selected as the run destination while unlocked.
- App Store Connect app `HexbaneDev` with Game Center group.

Open:

- **Sign in with Apple** — App ID capability + a *Sign in with Apple key* (`.p8`, server-side
  only) when the client bridge is written ([[ios-game-center-apple-auth]]).
- **Apple Distribution** certificate and App Store profile — let Xcode create them at the first
  archive/upload; nothing to do by hand.
- Push notifications, associated domains, etc. — not used; do not enable capabilities the binary
  does not claim (they are harmless, but every enabled capability regenerates profiles).
- A second Mac or CI needs the private key: export the identity from Keychain Access as `.p12`
  and import it there, or let each machine have its own Development certificate (a team may have
  several).

## Reading a failure

| Message | Meaning |
|---|---|
| *conflicting provisioning settings … automatically signed, but code signing identity Apple Distribution has been manually specified* | Release identity in the preset is empty/Distribution; set `Apple Development` |
| *No profiles for 'com.dev.hexbane' were found* / *Your team has no devices* | automatic signing could not build a development profile: phone not registered yet (run to it from Xcode, unlocked, trusted), or the Apple ID is not in Xcode → Accounts |
| *Provisioning profile doesn't include the com.apple.developer.game-center entitlement* | capability missing on the App ID → enable it, then let Xcode refresh the profile |
| *device was not, or could not be, unlocked* (devicectl launch) | unlock the phone; install works locked, launch does not |
| *Undefined symbol: _main* | you built for a simulator from the device export — see [[deploy-ios]] |
| App installs but Game Center says *not recognized* / signature errors | bundle id or team differs from the App Store Connect record, or the identity/id pair is wrong ([[ios-game-center-apple-auth]]) |

Inspect what a built app actually carries:

```bash
codesign -d --entitlements - path/to/hexbane.app          # entitlements in the signature
security cms -D -i path/to/hexbane.app/embedded.mobileprovision   # the profile (devices, expiry, entitlements)
security find-identity -v -p codesigning                  # identities in the keychain
```
