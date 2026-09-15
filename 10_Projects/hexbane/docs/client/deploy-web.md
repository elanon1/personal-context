---
type: project
project: Hexbane
area: client
status: experimental
created: 2026-09-14
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, web, godot, dotnet, deploy]
sources: ["client:hexbane.web/hexbane.web.csproj", "client:Scripts/web.sh", "https://2dog.dev/hosts/web"]
---

# Browser build with 2dog

The experimental build lives in `/Users/elanon/RiderProjects/hexbane-web`, branch
`codex/web-build`, based on `af927dd`. It is not merged into the normal client checkout.
Stock Godot 4 C# web export remains unsupported; this build uses the third-party
2dog runtime, preserving the existing C# game and Godot scenes.

## Existing-checkout rebuild

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

## Reconstruct from scratch

This procedure was recorded on 2026-09-15 against the actual scaffold, configuration,
and successful 2026-09-14 publish log. The first experiment's changes are still
uncommitted: checking out `codex/web-build` alone will not restore its host files.
The following pinned scaffold and adjustments also work as a recovery recipe when
that checkout is unavailable. A second full clean browser build has not been run
as part of documenting this procedure.

### 1. Isolate the source

Inspect before creating anything:

```bash
cd /Users/elanon/RiderProjects/hexbane
git status --short
git worktree list
git branch --list 'codex/web*'
df -h /
```

Reuse the existing web worktree when present. Otherwise create a new worktree with
an unused path and branch; the example below reproduces the original source base:

```bash
git worktree add -b codex/web-rebuild ../hexbane-web-rebuild af927dd
cd ../hexbane-web-rebuild
```

For a build of newer game code, replace `af927dd` with the intended commit.
Uncommitted main-checkout edits are not carried into a worktree. Incorporate only
the intended changes after establishing their provenance. Never reset an existing
checkout to recreate this experiment.

### 2. Install the separate pinned toolchain

This is the macOS installation used for the experiment; the dotnet installer
selects the machine architecture. Verify an existing installation before reusing
its directory. On another platform, use its supported .NET installation method.
The separate SDK avoids changing the established .NET 9 mobile build.

```bash
web_sdk="$HOME/.local/share/hexbane-dotnet10"
web_tools="$HOME/.local/share/hexbane-web-tools"
curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/hexbane-dotnet-install.sh
bash /tmp/hexbane-dotnet-install.sh --version 10.0.401 --install-dir "$web_sdk"
export DOTNET_ROOT="$web_sdk"
export PATH="$web_sdk:$PATH"
dotnet --version
dotnet workload install wasm-tools
dotnet workload list
```

Finish the workload installation before publishing. Install the CLI if missing:

```bash
dotnet tool install 2dog --version 4.7.2.84 --tool-path "$web_tools"
"$web_tools/2dog" version
```

If installed already, inspect `dotnet tool list --tool-path "$web_tools"` instead
of reinstalling blindly. The tested SDK was 10.0.401; its installed wasm packs
were 10.0.12. The original setup used `--channel 10.0`; this recipe pins the version
that command actually resolved to. Tool downloads still depend on upstream availability.

### 3. Generate the browser host

From the isolated game checkout, using the shell environment above:

```bash
"$web_tools/2dog" add --web --no-restore --dry-run
"$web_tools/2dog" add --web --no-restore
```

Review the dry-run scope. With CLI 4.7.2.84 it:

- switches `hexbane.csproj` to Godot.NET.Sdk 4.7.2 / net10.0;
- adds unsafe support, `LIBGODOT_ENABLED`, excludes `hexbane.web/**` from default
  game compilation and explicitly includes `hexbane.web/TwoDogWebBoot.cs`;
- creates `hexbane.web/` with Program.cs, generated bootstrap, page shell,
  `.gdignore`, host csproj, Directory.Build.props and global.json;
- creates root Directory.Build.props / Directory.Build.targets and global.json;
- migrates hexbane.sln to hexbane.slnx and adds Web and Linux export presets.

Generated package values should be `TwoDogVersion=4.7.2.84`,
`TwoDogNativesVersion=4.7.2.4`, `TwoDogGodotVersion=4.7.2`.
The host references `2dog.engine` and `2dog.browser-wasm` and roots the game assembly.
The generated global.json requests 10.0.100 with latestFeature roll-forward;
verify that the separate installation resolves it to 10.0.401 when reproducing
this snapshot. Check both root and host global.json if SDK resolution differs.

### 4. Apply the Hexbane-specific adjustments

Only in the isolated web checkout:

1. In project.godot set `[rendering] renderer/rendering_method.web="gl_compatibility"`.
2. Remove the `auth/google_client_secret` entry from the web copy of project.godot.
   Do not copy `.env`, client-secret JSON files, credentials or signing material
   into the web host's wwwroot. This build uses the existing email form.
3. In export_presets.cfg copy `exclude_filter` from the preset named **Android**
   into the preset named **Web**. Locate by name rather than assuming preset
   numbers. This preserves the project's mobile/SD resource policy. Verify that
   the Android preset still contains the expected exclusions when using newer code.
4. Add `<TrimmerRootAssembly Include="Nakama"/>` to the web host ItemGroup containing
   `<TrimmerRootAssembly Include="hexbane"/>`; keep the generated game and host roots.
5. Add `/hexbane.web/AppBundle/` to .gitignore. Keep hexbane.web/.gdignore.

For the original source base, the following idempotent helper performs those edits
and checks the expected configuration before writing:

```bash
python3 - <<'PYCONFIG'
from pathlib import Path
import re

project = Path('project.godot')
settings = project.read_text()
assert re.search(r'^renderer/rendering_method.web=', settings, re.M)
settings = re.sub(r'^renderer/rendering_method.web=.*$',
                  'renderer/rendering_method.web="gl_compatibility"', settings, flags=re.M)
settings = re.sub(r'^auth/google_client_secret=.*\n?', '', settings, flags=re.M)

presets = Path('export_presets.cfg')
parts = re.split(r'(?=^\[preset\.\d+\]$)', presets.read_text(), flags=re.M)
def preset_named(name):
    matches = [i for i, part in enumerate(parts)
               if re.search(r'^name="' + re.escape(name) + r'"$', part, re.M)]
    assert len(matches) == 1, f'Expected one {name} preset'
    return matches[0]
android, web = preset_named('Android'), preset_named('Web')
exclusion = re.search(r'^exclude_filter=(.*)$', parts[android], re.M)
assert exclusion and exclusion.group(1) != '""', 'Review Android exclusions first'
parts[web], count = re.subn(r'^exclude_filter=.*$',
    lambda _: 'exclude_filter=' + exclusion.group(1), parts[web], count=1, flags=re.M)
assert count == 1

host = Path('hexbane.web/hexbane.web.csproj')
xml = host.read_text()
anchor = '<TrimmerRootAssembly Include="hexbane"/>'
assert anchor in xml
if '<TrimmerRootAssembly Include="Nakama"/>' not in xml:
    xml = xml.replace(anchor, anchor + '\n        <TrimmerRootAssembly Include="Nakama"/>', 1)
ignore = Path('.gitignore')
ignored = ignore.read_text()
if '/hexbane.web/AppBundle/' not in ignored.splitlines():
    ignored += '\n/hexbane.web/AppBundle/\n'

project.write_text(settings)
presets.write_text(''.join(parts))
host.write_text(xml)
ignore.write_text(ignored)
PYCONFIG
```

### 5. Build and serve

The scratch recipe does not require Scripts/web.sh (it was handwritten after the
first successful export). These direct commands are sufficient:

```bash
dotnet build hexbane.csproj
mkdir -p verification/web
dotnet publish hexbane.web -c Release > verification/web/publish.log 2>&1
```

Check publish's exit status and the full log; `tail` alone does not establish
success. 2dog restores the native editor/helper packages, imports resources,
exports the pack and links the wasm runtime. A clean import can take longer than
the original run, which reused an APFS clone of `.godot/imported`. Reusing that
cache is optional and not required for reconstruction. Never share a writable
`.godot` directory between concurrent checkouts.

Then, in a terminal that can stay running:

```bash
python3 -m http.server 8067 --bind 127.0.0.1 --directory hexbane.web/AppBundle
```

Use another unused port when occupied. From another terminal verify
`curl -I http://127.0.0.1:8067/`, then open that URL in a real browser. Opening
index.html as a file is not equivalent to serving it over HTTP.

### 6. Validate browser behavior

Wait for a visibly finished load. Verify the actual login screen and input;
choose **Production** for the public backend or the intended reachable development
server. Have the user perform authentication when credentials are unavailable.
Exercise Training and distinguish that observation from an authenticated online
match. The original credential/Training navigation was performed by the user and
was not captured as a deterministic UI recipe; inspect current controls rather
than guessing those steps. Check audio after a user gesture.

For failures, distinguish compile/link errors, resource import errors, browser
runtime exceptions, and network/auth failures. Capture the relevant log or browser
console evidence before changing code. The original export contained TileSet and
editor-shutdown errors that did not prevent visible training; this is historical
evidence, not an instruction to ignore similar errors in a new build.

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

## Reusable Codex skill

Invoke `$hexbane-web-build`. Its installed entrypoint is
`/Users/elanon/.codex/skills/hexbane-web-build/SKILL.md`; it routes to this note
so the reconstruction procedure remains maintained in the project vault.

## Source of truth in code

Paths below refer to the experimental checkout until integrated:
- client:hexbane.web/hexbane.web.csproj and Program.cs — web host and trimming roots
- client:hexbane.csproj, Directory.Build.props, global.json — isolated toolchain
- client:project.godot and export_presets.cfg — renderer and pack configuration
- client:Scripts/web.sh — build and loopback server
