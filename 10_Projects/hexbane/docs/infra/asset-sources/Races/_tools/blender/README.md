# Race animation pipeline: Mixamo rig → Blender → sprite sheets → Godot

Characters are 3D models from Tripo, rigged in Mixamo (65 bones, fingers included), animated
with Mixamo clips (or hand-edited in Blender) and **pre-rendered to 2D sprite sheets**. The game
stays 2D; `RaceAnimationPreview` plays `Resources/Races/<race>/animation/frames.tres`.

All six races share the same `mixamorig:*` skeleton, so every animation works on every race
without retargeting. Blender is at `/Applications/Blender.app/Contents/MacOS/Blender` (5.2).

## Inputs per race

| What | Where | Notes |
|---|---|---|
| Rigged character FBX | `ArtSource/Races/Models/<race>/tripo_convert_<id>.fbx` | Tripo model converted to the Mixamo rig. Keep its `<same name>.fbm/` folder next to it: it holds the 4K texture; without it the model renders magenta. Tripo's download has an *unrigged* FBX at the root and the rigged one inside `Magic Spell Pack/` — take the latter. |
| Animation FBXs | `ArtSource/Races/Models/Magic Spell Pack/*.fbx` | Mixamo downloads "without skin", 30 fps, shared by every race. Filenames are the Mixamo clip names. A race that ships its own clips (`~/3D/<race>/Magic Spell Pack/standing idle.fbx` present) uses those instead — a re-rigged model comes with a pack built for its rig, and mixing rigs bends joints the wrong way. |

All six races (`human`, `elf`, `dark_elf`, `shadow`, `gnome`, `orc`) are converted; their source
downloads sit in `~/3D/<race>/`. For a new race: upload its Tripo GLB to Mixamo (or use Tripo's
Mixamo conversion), download the rigged FBX **with skin** plus its texture, drop the folder into
`~/3D/`, and add the race to the `RACES` map in `ArtSource/Races/Tools/build_races.sh`.

## Shortcut: build every race in one go

```bash
ArtSource/Races/Tools/build_races.sh            # all races, ~4 min each
ArtSource/Races/Tools/build_races.sh orc gnome  # just these
```

It runs steps 1, 3 and 4 below for the 13 clips of the Magic Spell Pack and writes each race's
`frames.tres` plus the smaller `frames_sd.tres`. The manual steps stay useful when you hand-edit an animation (step 2).

## Step 1 – assemble: character + animations → one .blend

```bash
B=/Applications/Blender.app/Contents/MacOS/Blender
T=ArtSource/Races/Tools/blender
P="ArtSource/Races/Models/Magic Spell Pack"
$B -b -P $T/assemble_mixamo.py -- \
  --character "$P/tripo_convert_23c7e892-2944-488b-80de-a1d6b33319cf.fbx" \
  --anim "idle=$P/standing idle.fbx" \
  --anim "spell_throw=$P/standing 1H cast spell 01.fbx" \
  --out ~/hexbane-archive/Models-2026-09-03/blend/human_mixamo.blend
```

`godot_name=path.fbx` — the left side becomes the action name in Blender **and** the
animation name in Godot. Game names: `idle`, `spell_throw`, `meditation`, `victory`, `dead`,
`damage_taken` (`idle` and `meditation` loop). Add as many extra clips as you like
(`cast_2h=…`, `attack_1h_01=…`) — they are only rendered if you ask for them in step 3.

The hips translation is rescaled to the character's height so short races keep their feet
on the ground.

## Step 2 – (optional) edit or create an animation in Blender

Open the .blend from step 1 in the Blender UI. The armature has one action per clip.

1. Select the armature → **Pose Mode**. Dope Sheet → **Action Editor**, pick the action
   (e.g. `spell_throw`). The timeline range follows the action; Auto Keying is on.
2. Mixamo clips carry a key on every frame. Before editing, thin them: Graph Editor →
   select all → Key → **Clean Keyframes** (or Decimate to ~10 %). Now only the extremes remain.
3. Edit: rotate bones on an existing key (Auto Key records it), drag keys in the Dope Sheet to
   retime, scale keys (S) to speed up / slow down. Bones worth touching: `mixamorig:Hips`,
   `Spine`, `Spine1/2`, `Neck`, `Head`, `Left/RightShoulder`, `Arm`, `ForeArm`, `Hand`, the
   finger chains `HandThumb1-3` … `HandPinky1-3`, `UpLeg`, `Leg`, `Foot`.
4. New animation from scratch: Action Editor → **New**, pose the first key on frame 1 (copy the
   idle pose: select all bones in idle, Ctrl+C, switch action, Ctrl+V), then key poses every
   6–10 frames. For a looping clip copy the first pose onto the last frame.
5. Ctrl+S. The .blend is the source of truth; re-running step 1 **overwrites** it, so if you
   have hand edits, add new clips by importing the FBX manually (File → Import → FBX, then
   rename the action) or copy your edited action into the regenerated file (File → Append).

Because the skeleton is shared, an action edited here can be reused on another race:
File → Append → `<other>.blend` → Action, then assign it in the Action Editor.

## Step 3 – render sprite frames

```bash
$B -b ~/hexbane-archive/Models-2026-09-03/blend/human_mixamo.blend -P $T/render_sprites.py -- \
  --race human --anim idle --anim spell_throw --fps 15 --size 1024 --angle 20
```

- One orthographic camera per race, framed on the union of all requested animations, so
  every clip shares scale and ground line. `--angle 0` is a pure side view, `20` a slight
  3/4 towards the camera (used for the Human). Character faces screen-right.
- `--fps` must divide the scene fps (30 for Mixamo): 15, 10 or 6. The log prints the value to
  pass to step 4.
- Frames go to `Resources/Races/<race>/animation/<anim>/render/f_####.png` (transparent).
- `--glb file.glb` renders a Tripo GLB directly (Tripo's own rig/animations) instead of a .blend.

## Step 4 – pack into frames.tres

```bash
python3 ArtSource/Races/Tools/build_frames_from_renders.py human --fps 15 --height 512 --cols 8 \
  --anims idle,spell_throw
```

Writes `<anim>_sheet.png` per animation and `Resources/Races/<race>/animation/frames.tres`
with every animation found. Frames are cropped to the race's union alpha bounds (no jitter
between frames) and scaled to `--height`. Add `--mirror` if a race must face left.

### Two resolutions, one per build

The renders are packed twice, so a desktop build can keep the crisp sheets while a phone ships
a quarter of the bytes:

```bash
python3 ArtSource/Races/Tools/build_frames_from_renders.py human --fps 15 --height 512 --cols 8 --anims …
python3 ArtSource/Races/Tools/build_frames_from_renders.py human --fps 15 --height 320 --cols 8 --anims … --variant sd
```

`--variant sd` writes `<anim>_sheet_sd.png` + `frames_sd.tres` next to the full-size set;
`build_races.sh` runs both passes. The client picks one in
`RaceAnimationPreview.GetRaceFramesPath`: small on mobile/web, full-size elsewhere, overridable
with the project setting `hexbane/graphics/race_sprites` (`auto` | `hd` | `sd`). Each export
preset drops the set it does not use (`exclude_filter` in `export_presets.cfg`), and the animator
scales whatever it loads to the same on-screen height, so the two look identical apart from sharpness.

The raw `render/` folders carry a `.gdignore` — they are intermediates worth ~2 GB and must never
reach an export.

## Step 5 – check in Godot

Open `Game/ScenesV3/Dev/RaceAnimTest.tscn` and run it (F6): pick the race, click an
animation. The scene uses the same `RaceAnimationPreview` component as the game.

## Meditation (all six races)

Each race now has three skeletal actions and matching HD/SD clips:

- `meditation_enter`: 13 frames at 15 fps, easing from the idle pose to joined hands.
- `meditation`: 60 frames at 15 fps, a four-second breathing loop.
- `meditation_exit`: 13 frames, the reverse of the entrance.

Runtime plays entry in 0.18 s and exit in 0.16 s, preserving the original source
keys while making short meditation actions readable. Blue absorption VFX and ground
seals use the bone positions in `meditation_tracks.tres`, exported with
`export_meditation_tracks.py` (included in `build_races.sh`).

Editable, texture-packed sources live in
`ArtSource/Races/Models/<race>/meditation/<race>_meditation.blend`. These folders have
`.gdignore` so Blender sources are excluded from game exports. The timeline shows the
loop; switch Actions to edit the entrance or exit. All 65 bones have keyframes.

To regenerate from the original assembled race (also run by `build_races.sh`):

```bash
/Applications/Blender.app/Contents/MacOS/Blender -b ~/hexbane-archive/Models-2026-09-03/blend/shadow_mixamo.blend \
  -P ArtSource/Races/Tools/blender/create_meditation.py -- --race shadow --render
python3 ArtSource/Races/Tools/pack_meditation.py shadow
```

The generator derives hand placement and palm orientation from the race's arm and
finger bones. It preserves the original thirteen-action camera bounds, lights and
idle foot positions. The packer recovers the original crop and verifies the existing
idle atlas pixel for pixel before appending the new clips. Existing cast sheets and
palm tracks remain compatible. Re-running the generator replaces authored meditation
actions; save manual edits separately before doing so.

`Player` passes the authoritative meditation state to `RaceSpriteAnimator` for both
duelists. It plays entry → loop → exit → idle, reverses an interrupted transition from
the current pose, and lets spell casting interrupt immediately. Repeated snapshots
preserve animation progress. Paralysis, poison, death and match end stop meditation.

Run `Game/ScenesV3/Dev/MeditationTest.tscn` to check all six races in HD and SD,
including cancellation, casting, recovery, facing, looping and player-state wiring:

```bash
dotnet build hexbane.csproj
/Applications/Godot.app/Contents/MacOS/Godot --headless --fixed-fps 60 --path . \
  Game/ScenesV3/Dev/MeditationTest.tscn
```

## Repeat for another race / animation

- New animation for an existing race: add `--anim name=clip.fbx` to step 1 (or edit in step 2),
  then steps 3–4 with the new name added to `--anim` / `--anims`.
- New race: get its Mixamo-rigged FBX + `.fbm`, run steps 1, 3, 4 with `--race <race_id>`.
  The same `--anim` list works unchanged because the skeleton is identical.

## Legacy: Tripo rig

`setup_edit.py` prepares a Tripo GLB (41-bone Tripo rig, no fingers) for editing one of its
preset animations. Kept for reference; the Mixamo route above is the one in use.

## In-game use (arena)

`Game/ScenesV3/GameHud/ArcaneDuel/Components/Player.tscn` no longer carries the old Skeleton2D rig:
it holds a `RaceSpriteAnimator` (`Game/ScenesV3/Components/RaceSpriteAnimator.cs`) that loads
`Resources/Races/<race>/animation/frames.tres` — falling back to `human` for races without frames —
loops `idle`, and on opcode 21 plays the cast clip chosen by
`CastAnimationResolver` (`Game/ScenesV3/Components/CastAnimationResolver.cs`):

1. `spell.animation` from the server, if that clip exists in the race's frames;
2. otherwise `attack_1h_01` (the project default);
3. otherwise `spell_throw`.

The clip is time-scaled to the cast: speed = clip length / `adjusted_casting_time` (falls back to
`spell.casting_time`), clamped to 1×–6×; the last frame holds until the cast is accepted or fails,
then idle resumes. Match payloads carry `casting_time` in **seconds** (plus the DEX/race-adjusted
`adjusted_casting_time`), unlike the spellbook RPCs' `cast_time` in ms — `Spell.CastDurationSeconds`
hides the difference.

Both duelists render as their own race: the server puts `race_id` on the public and private player
views (`player_state.ToPublicView` / `ToPrivateView`), and `Player.OnGameLoad` loads the animator
from the view — the local character's race is only the fallback for a server that does not send it.
The AI opponent gets a random race at match init (`ai_match/init.go`, `randomBotRace`).

Server side, to drive a different clip per spell, add `animation: <clip name>` to the spell yaml
(`spell_system.Spell`) and serialise it as `"animation"`; nothing else is needed on the client.
