---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-10
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, loading, ui, mobile]
sources: ["client:Game/Autoloads/SceneManager.DuelLoading.cs", "client:Game/ScenesV3/Loading/DuelLoadingScreen.cs"]
---

# Duel loading screen

The arrangement → arena handoff uses a persistent `DuelLoadingScreen` CanvasLayer owned by SceneManager (layer 400), replacing the black fade on the `normal_game` route. Other scene transitions keep their existing presentation. The user explicitly requested a new blue/emerald illustration and a different motif; this loading screen is an intentional exception to the shared warm menu-background rule.

## Presentation

- Generated drowned crystal sanctuary in navy, cyan and emerald: `Resources/Images/Loading/emerald_sanctuary.png` (1672×941, about 2.3 MiB). No text is baked into the bitmap.
- Native UI: HEXBANE title, Entering the arena, current loading stage, mint progress bar, breathing diamond and one randomly selected gameplay tip. Seven short tips cover queuing, poison/meditation, standards, opponent casts, arrangement, barriers and mana reserves. The selected tip remains fixed for the entire load and viewport resize.
- Background uses cover aspect ratio. Content width is capped at 820 logical units and shrinks within side margins; wrapped titles/tips and spacing adapt to viewport height. Minimum tip/status text is 16 logical units. Mobile display safe-area insets constrain content, not the background. Retry has a 48-unit touch target.
- No percentage text: the bar is a weighted completion indicator, not bytes downloaded or elapsed time. The first 85% aggregates Godot resource-loading progress; scene initialization is 90%, first rendered arena is 100%. A warm-cache load keeps the screen for a minimum 0.9 s to avoid a flash, followed by a 0.3 s reveal. The server still waits for presentation readiness, so this does not consume countdown time.

## Loading and lifecycle

The illustration is drawn before requesting heavyweight resources. SceneManager uses `LoadThreadedRequest/GetStatus/Get` for the packed arena, selected arena texture, reference HUD art and available/fallback race frames. References are retained through scene `_Ready`, reducing synchronous asset loading there. Scene instantiation and GPU initialization still run on Godot's main thread and may briefly pause animation; the illustrated frame remains visible.

The overlay survives removal of the old scene and stays until `SceneChanged`, a rendered frame and final fade. Existing `ReferenceHud.SendReadyAfterPresentation` waits for transition completion before acknowledging the server. Navigation requested during loading is queued and cancels the arena handoff. Every accepted threaded request is collected even after cancellation/failure; otherwise Godot retains failed tokens and retries cannot recover. Failures preserve the current scene, keep the illustration visible and offer Try again; subsequent normal navigation removes the error overlay.

No backend changes or deployment in this task. Source is built locally; installed mobile clients need a new build.

## Verification

- `dotnet build hexbane.csproj --no-restore`: 0 errors, 11 existing warnings.
- `Dev/DuelLaunchVerification.tscn`: regression failed before the overlay existed; passes with persistent overlay, monotonic progress, stable tip, actual lobby replacement, overlay cleanup and preserved countdown.
- `Dev/LoadingScreenVerification.tscn`: rendered checks pass at 1920×1080, 1360×612, 960×432, 844×390, 390×844 and 2560×1080; title/status/progress/tip/brand fit, tip lines remain visible, resizing preserves the tip and progress never decreases. Retry target ≥48.
- `Dev/LoadingLifecycleVerification.tscn`: deliberately corrupts a temporary `user://` scene, repairs it and presses Retry; confirms recovery, cancellation navigation and a later successful arena load. Expected parser/load errors are part of this test. Temporary fixture is removed and route restored.
- Rendered evidence: `verification/loading-screen/{desktop,mobile,portrait}.png`. Standalone F6 preview: `Game/ScenesV3/Loading/DuelLoadingPreview.tscn`.
- Independent review found uncollected canceled/failed threaded requests; fixed and verified by the lifecycle harness. No remaining review blocker. Physical device touch/notches and mobile export remain manual validation.

## Image generation

Built-in ImageGen. First warm portal concept was rejected and is not shipped. Final image is the blue/emerald crystal sanctuary; source output copied into the repository, not referenced from Codex's generated-image directory.

Final prompt:

> Use case: stylized-concept. Brand NEW loading background for a dark fantasy spell-duelling game. Landscape 16:9. Completely different motif from a gate or portal: a mysterious drowned sanctuary beneath a vast cavern ceiling, a fractured faceted magical crystal levitates above a circular ancient stone dais rising from still dark water. Streams of cyan and emerald magical energy subtly spiral around the crystal, crystalline shards hover nearby, distant broken pillars and submerged ruins dissolve into deep turquoise mist, faint bioluminescent algae. Moody painterly premium dark fantasy game environment concept art, hand-painted texture, elegant restrained magic, richly layered depth, not photorealistic and not sci-fi. Palette strictly deep midnight blue, petrol teal, luminous emerald green, icy cyan highlights; no orange, no gold, no fire, no warm sunset. Centerpiece in upper-middle of frame, exquisite silhouette, mysterious and calm rather than explosive. Lower quarter remains quiet dark blue water with soft reflections to hold readable live progress bar and tip. Beautiful readable midtones: atmospheric darkness but not a black screen. No humans, no statues, no gates or archways, no rings of glyphs, no text, no letters, no logo, no watermark, no interface, no progress bar. Standalone production illustration.

## Source of truth in code

- `client:Game/Autoloads/SceneManager.cs`, `SceneManager.DuelLoading.cs` — route, threaded loading and persistent overlay lifecycle.
- `client:Game/ScenesV3/Loading/DuelLoadingScreen.cs`, `DuelLoadingPreview.tscn` — responsive screen, progress and tip pool.
- `client:Resources/Images/Loading/emerald_sanctuary.png` — shipped generated illustration.
- `client:Game/ScenesV3/Dev/{DuelLaunchVerification,LoadingScreenVerification,LoadingLifecycleVerification}.{cs,tscn}` — verification.
