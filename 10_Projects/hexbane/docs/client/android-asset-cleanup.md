---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-13
updated: 2026-09-13
verified: 2026-09-13
tags: [hexbane, android, assets, export, audit]
---

# Android asset cleanup

## Measured result

The user's approximately 600 MB artifact was identified as the 606.83 MiB release AAB. The existing root debug APK was 1002.18 MiB. These are different package formats and must not be compared as if they were identical. MiB means 1,048,576 bytes.

| Artifact | Before | After | Reduction |
|---|---:|---:|---:|
| Android Play release AAB | 606.83 MiB | 400.80 MiB | 34.0% |
| Android universal debug APK | 1002.18 MiB | 642.22 MiB | 35.9% |

New local artifacts:
- `/Users/elanon/RiderProjects/export/android/hexbane-v10-asset-cleanup.aab`
- `/Users/elanon/RiderProjects/export/android/hexbane-asset-cleanup.apk`

Old artifacts are retained for comparison. Version/signing/architecture settings are unchanged; no upload or install was performed.

## Cause

Every export preset used `all_resources` with only a few exclusions. Godot therefore packed imported resources regardless of whether gameplay referenced them: source-model textures, concept/reference art, screenshots, old UI packs, unused fonts and music, and MCP/development files. Four CPU architectures also contribute native engine and .NET overhead. The source tree's multi-gigabyte size is not the same as the compressed bundle size.

## Removed and excluded

- Removed **4589 files**, including matching import metadata and obsolete pack sidecars, totaling **233.97 MiB** from the working tree. This is source-file size, not the exact packaged saving.
- Unused UI asset packs, old environment concepts, unused font variants, obsolete frame/logo/concept art, unused menu music/stems and the unused root `unnamed.jpg` were moved out of the project.
- Recoverable archive: `/Users/elanon/hexbane-archive/unused-assets-2026-09-13`. `verification/asset-cleanup/removed.json` records every path, byte count and SHA-256. No broad git cleanup/revert/staging was performed; earlier worktree changes were preserved.
- `Resources/Races/3d`, `_tools` and `_tpose` are still authoring inputs; they remain in place with `.gdignore` and export exclusions. Do not delete them as though they were unused runtime art.
- All presets exclude verification output, reference art, docs, MCP server, scripts, tests, the exported `apk.app` tree, authoring inputs, dev scenes and old arena scenes. `Resources/Backgrounds` remains for legacy/dev previews but is excluded from runtime exports. The VfxTest route remains available.
- `verification/.gdignore` prevents future generated screenshots/audio artifacts from becoming imported/exported resources.
- All fourteen spell textures use Godot `process/size_limit=512`; the full-resolution source PNGs are preserved. The two reference icons remain byte-for-byte unchanged. See [[spell-icon-art-direction]].
- Mobile SD/desktop HD selection remains intact. No used race animation was removed or recompressed.

## How unused resources were identified

Scanned code/scenes/resource references, followed resource dependencies, resolved import UIDs referenced by presets/project settings, and checked filename assembly in C# rather than relying only on literal full paths. Explicitly preserved dynamic race, avatar, spell, effect, cast-preset and audio loaders. Whole retired directories were checked independently so unrelated generic filenames such as `star.png` did not keep obsolete packs alive.

`verification/asset-cleanup/audit.py` is the task's discovery snapshot; its candidates were reviewed before removal. It is not a proof that an arbitrary future dynamic loader is safe to prune. The removed manifest is the authoritative record of this task.

## What remains in the release AAB

Compressed ZIP-entry contributions (small package metadata overhead is additional):

| Category | MiB |
|---|---:|
| Six used race asset sets, primarily SD animation sheets | 167.1 |
| Native libraries across four CPU architectures | 106.2 |
| Packaged .NET assemblies/runtime assets | 46.2 |
| Used music | 13.7 |
| Fourteen spell icons after 512 px import | 4.7 |

The remainder is active UI, arenas, portraits, fonts, spell sound effects and package metadata. A universal debug APK is larger because it includes all ABIs and its native libraries are stored without ZIP compression. Reducing used animation resolution or supported architectures would be a separate quality/compatibility decision.

## Verification and limits

- Final .NET build: exit 0; zero errors.
- Godot import and both final exports: exit 0.
- MobileLayoutVerification: zero overflows at 1360×612.
- GameplaySandboxVerification: PASS, 14 icons/casts, reset cancellation, reverse reflection/dodge, meditation/audio and held statuses.
- Godot icon preview: all 14 textures loaded; visually reviewed at 160 px and 48 px after the final 512 px import change.
- Final package audit: both new packages contain all 14 expected icons and all six SD race sets, with **zero** hits for excluded source models, reference/verification material, retired asset packs or HD sheets.
- Retained code/resource scan found zero direct references to removed assets; preserved icon SHA-256 checks pass.
- Existing Godot shutdown warnings (`ObjectDB/resource` and `EditorSettings`) occur in logs; they are distinct from the passing test/export results.
- No physical Android startup, full live-Nakama duel or Google Play upload was performed in this task.

## Source of truth in code

- client:export_presets.cfg
- client:hexbane.csproj
- client:Resources/Races/3d/.gdignore
- client:Resources/Races/_tools/.gdignore
- client:Resources/Races/_tpose/.gdignore
- client:verification/.gdignore
- client:Resources/Spells/*/*.png.import
- client:verification/asset-cleanup/summary.json
- client:verification/asset-cleanup/removed.json
- client:verification/asset-cleanup/check_packages.py
- client:verification/asset-cleanup/reference-check.json
- client:verification/spell-icons/manifest.json
