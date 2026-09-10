---
type: project
project: Hexbane
area: plans
status: complete
created: 2026-09-09
updated: 2026-09-09
verified: 2026-09-09
---
# Godot 4.7 migration

Requested: migrate existing working tree to installed Godot 4.7 .NET and remove API 36 default-35 export warning.

1. Back up touched config/scripts and old generated Android template outside repo, preserving unrelated work.
2. Install matching official templates; update project and test Godot SDK and engine feature version; update default deployment executable.
3. Replace generated Android build template while retaining custom project/plugin configuration. Build release Android Play AAB and inspect target SDK/signature.
4. Adapt iPhone simulator script to matching 4.7 template capabilities; do not mix 4.5.2 native engine with 4.7 managed API.
5. Verify C# build, 4.7 project loading/startup and exports; update vault contracts and session log.

No game-design changes or automatic scene-wide resaving. No commits or uploads requested.

## Results

Main and Auth C# builds: zero errors; 17 auth checks passed. Signed AAB built, bundletool confirmed compileSdk36/targetSdk36/versionCode5 and no higher-than-default warning. iOS ARM64 engine rebuilt for4.7; simulator build/install/launch succeeded and authentication UI rendered. Godot4.7 editor opened for the project. Full Android16 gameplay/Google Play acceptance not tested. Existing C# warnings and headless EditorSettings shutdown message remain.
