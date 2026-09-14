---
type: project
project: Hexbane
area: client
status: experimental
created: 2026-09-14
updated: 2026-09-14
verified: 2026-09-14
tags: [hexbane, web, godot, dotnet, deploy]
sources: ["client:hexbane.web/hexbane.web.csproj", "client:Scripts/web.sh", "https://2dog.dev/hosts/web"]
---

# Browser build with 2dog

The experimental build lives in `/Users/elanon/RiderProjects/hexbane-web`, branch
`codex/web-build`, based on `af927dd`. It is not merged into the normal client checkout.
Stock Godot 4 C# web export remains unsupported; this build uses the third-party
2dog runtime, preserving the existing C# game and Godot scenes.

## Toolchain and commands

- .NET SDK 10.0.401 and wasm-tools installed separately at
  `/Users/elanon/.local/share/hexbane-dotnet10`.
- 2dog CLI 4.7.2.84 at `/Users/elanon/.local/share/hexbane-web-tools`.
- Godot.NET.Sdk 4.7.2; 2dog.engine 4.7.2.84; native packages 4.7.2.4.
- Run from the experimental checkout:

```bash
./Scripts/web.sh build
./Scripts/web.sh serve
```

Open http://127.0.0.1:8067. `PORT` changes the local server port;
`HEXBANE_WEB_DOTNET_ROOT` can select another SDK installation. The server binds
only loopback. Output is `hexbane.web/AppBundle/`; build logs from the first
successful publish are in ignored `verification/web/publish.log`.

## Configuration

The scaffold uses .NET 10, a generated browser bootstrap and a separate host.
Web selects `gl_compatibility` / WebGL 2 and uses the mobile asset exclusion list.
RaceAnimationPreview already selects SD animations for the web feature.
Nakama is explicitly rooted for managed trimming because TinyJson uses reflection.
The desktop OAuth secret is absent from the experimental project settings;
the existing email form is available. A browser-specific Google OAuth flow has
not been implemented. Select Production for the public backend; Local is still
the default server selection.

## Verified and remaining

- C# build: exit 0, zero errors, eight warnings.
- Full browser publish: exit 0, generated wasm, runtime, resource pack and page.
- Local HTTP index: 200 OK.
- Actual Dia browser: rendered the normal login screen. The user interacted with
  the browser during validation; a subsequent visual observation showed active
  Training with arena, race sprites, spell bar, changing HP/mana and combat text.
  The assistant did not perform or record the intervening credential steps.
- This proves browser startup and visible training, not an online PvP match,
  full tutorial completion, fresh registration, or Google authentication.
- Export logs contain existing TileSet errors and editor shutdown warnings;
  publish succeeds, but these diagnostics were not audited in this experiment.
- The raw resource pack is about 292 MiB and wasm about 45 MiB. The output directory
  is about 980 MiB including Brotli/gzip siblings. Download size still needs work
  before public hosting. No public deployment was performed.
- Threading/platform restrictions, all online flows, other browsers and mobile web
  remain to be tested. See https://2dog.dev/hosts/web for runtime limitations and
  https://docs.godotengine.org/en/latest/tutorials/export/exporting_for_web.html
  for the official exporter status.

## Source of truth in code

Paths below refer to the experimental checkout until integrated:
- client:hexbane.web/hexbane.web.csproj and Program.cs — web host and trimming roots
- client:hexbane.csproj, Directory.Build.props, global.json — isolated toolchain
- client:project.godot and export_presets.cfg — renderer and pack configuration
- client:Scripts/web.sh — build and loopback server
