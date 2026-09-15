# Duel HUD plates — ChatGPT prompts

Painted frames for the in-match HUD, matched to `reference/fight/game.png`.

The HUD works without these: `HudArt.PaintedPlate()` returns null when a file is missing and
`HudPanel` falls back to drawing the frame. Drop a PNG in beside this file and it is picked up
on the next run — **no code change**.

| file | drawn where | aspect | delivered |
|---|---|---|---|
| `plaque.png` | player / enemy card, top corners | 3.329 : 1 | ✅ |
| `timer.png` | clock banner, top centre | 6.505 : 1 | ✅ |
| `portrait_frame.png` | medallion around the avatar | 1 : 1 | ✅ |
| `slot.png` | all ten spell slots | 1 : 1 | ✅ |

The aspects above are **measured on the delivered art**, and `GameHudScreen` is fitted to
them (`PlaqueAspect`, `TimerAspect`) rather than the other way round — stretching a plate off
its own aspect smears the filigree and stretches the finial. If you regenerate a plate at a
different aspect, update the matching constant.

Only `plaque.png` is drawn with the notch on the **right**; the enemy card reuses the same
file mirrored, so do not generate a second one.

The reference splits the spell slots into a cyan group and a magenta group. We dropped that:
both groups are the local player's own spells, and a neon border read as a sticker next to the
painted brass. One brass slot serves all ten.

Generated 2026-09-01 in ChatGPT (Volt workspace):
https://chatgpt.com/c/6a9717d3-f5ac-83ed-87dd-e79f25b16764

## How to run this

1. New ChatGPT conversation, newest image-capable model, image generation on.
2. **First message:** paste `reference/fight/game.png` with the STYLE block below, so every
   later asset is anchored to the same art.
3. Then one asset prompt per message, each prefixed with the STYLE block.
4. Hand the results back here — the background gets keyed out and the files installed.

### Measure the aspect on the FRAME, not the canvas

The first pass asked for "three times as wide as it is tall" and the model applied that to the
whole canvas *including* the green margin. Once the green was keyed away the plate itself came
out 5.04 : 1 — unusable at the intended proportion. Say **"measured on the outer edge of the
brass frame, ignoring the green"** explicitly, and verify by keying and re-measuring before
accepting.

### Why a green background and not transparency

The model does not produce reliable alpha; it fakes it, or leaves a soft halo that shows up
as grey fringing over the arena. So every plate is generated on a **flat solid chroma** far
from the brass/indigo palette and keyed out afterwards. This is the same route the
`race-maker` skill uses, for the same reason.

---

## STYLE  (prepend verbatim to every asset prompt)

```
STYLE: Painted dark-fantasy game UI ornament, matching the attached reference screenshot.
Antique brass and tarnished gold metalwork, two-tone: dark bronze #6b5326 on the outer edge,
bright polished gold #e8d191 on the raised inner edge, so the frame reads as bevelled metal
rather than a flat outline. Restrained gothic filigree — small pointed finials, corner
brackets, fine engraved lines — never busy or floral. Interior of the frame is deep
near-black indigo #0b0916 and completely EMPTY: no text, no numbers, no bars, no icons, no
portraits, no runes inside the panel area. Slight age and wear on the metal, faint warm
rim-light, no strong bloom.
BACKGROUND: flat SOLID chroma green #00FF6A filling the whole canvas around the object. No
gradient, no vignette, no shadow, no glow, no soft or feathered edges — the ornament must end
in a hard clean edge against the green so it can be keyed out. Nothing else in the image.
VIEW: perfectly flat straight-on 2D orthographic, no perspective, no tilt, no drop shadow.
```

---

## 1 — `plaque.png`  (player card frame)

```
<STYLE>

Draw ONE horizontal ornate frame plate for a player status card, centred in the canvas with a
wide green margin all around it. Proportions: three times as wide as it is tall.

The LEFT end is a squared corner with small chamfers. The RIGHT end tapers inward to a
shallow arrow notch that points back into the plate — a chevron cut about a fifth of the
height deep at the vertical middle — finished with a small pointed brass finial.
All four corners carry short bracket ornaments.

The interior is one large empty dark indigo field spanning almost the whole plate. Leave it
completely bare — the game draws the portrait, name, health bar, mana bar and effect icons
into it at runtime. Do not draw any of those.
```

## 2 — `timer.png`  (clock banner)

```
<STYLE>

Draw ONE small horizontal banner plate for a match clock, centred with a wide green margin.
Proportions: three and a half times as wide as it is tall.

Both ends taper inward to shallow arrow notches pointing into the banner, each capped with a
pointed brass finial. A thin engraved brass line runs just inside the top and bottom edges.
Small bracket ornaments at all four corners.

The interior is empty dark indigo — the clock digits are drawn by the game. Do not draw any
numbers, digits or colons.
```

## 3 — `portrait_frame.png`  (avatar medallion)

```
<STYLE>

Draw ONE circular medallion frame for a character portrait, centred in a square canvas with a
wide green margin.

A heavy brass ring with a beaded inner edge and four small pointed finials at the top, bottom,
left and right. Below the ring, a small brass shield emblem overlapping the lower rim.

The centre of the ring is EMPTY dark indigo — the game composites the avatar inside it. Do not
draw a face, a head, a silhouette or any figure.
```

## 4 — `slot.png`  (all spell slots)

```
<STYLE>

Draw ONE square spell-slot frame, centred in a SQUARE canvas with a wide green margin.

A rounded square with all four corners cut at 45 degrees, small bracket ornaments at each cut
corner. The border is the SAME antique brass as the player status plate — dark bronze on the
outer edge, bright polished gold on the raised inner edge, bevelled, slightly worn. It has to
look like it was cut from the same sheet of metal as the status plate and the clock banner.
No glowing line, no neon, no coloured border anywhere.

Measured on the outer edge of the brass frame and ignoring the green margin, the frame must be
exactly square, 1:1.

The interior is empty dark indigo with a very faint eight-pointed arcane star barely visible in
the centre — a whisper, not a feature, because a spell icon is drawn over it at runtime.
```

---

## Checking a result before accepting it

- The background is one flat green with **hard** edges — no halo, no soft fade, no shadow.
- The interior is empty. Any baked-in text, bar or number makes the plate unusable, because
  the live values are drawn on top and would double up.
- The proportions match the table above. A plate generated square and then stretched to 3:1
  smears the filigree.
- `plaque.png` notches on the **right** only.

## If the download button stops working

Late in the session ChatGPT's own Download button silently stopped saving anything. Forcing it
from the page works and keeps full resolution — draw the image onto a canvas, `toBlob`, then
click a synthetic `<a download>`. Do not round-trip the image through base64: it costs a large
amount of context for no gain.

---

# Batch 2 — reference/fight/game.png, 2026-09-04

The second reference pass replaced the square slots and the solid status plaque with **round**
slot rings on gold rails, an oval portrait medallion and thin chevron bar frames. The pieces
below are what the new HUD needs. `plaque.png` and `slot.png` from batch 1 are superseded.

| file | drawn where | delivered aspect (w/h) |
|---|---|---|
| `slot_round.png` | the 6/7 drafted spell slots | 1.000 |
| `slot_round_locked.png` | a slot the level has not unlocked — padlock, tarnished | 1.000 |
| `slot_round_unavailable.png` | a spell that cannot be cast right now — steel, dashed light | 1.000 |
| `slot_round_large.png` | Magic Arrow / Mirror Reflection | 1.000 |
| `standard_bracket.png` | crescent + lozenge on the outer flank of a standard ring (mirrored for the right one) | 0.331 |
| `rail_oval.png` | oval rail the LEFT cluster sits inside | 1.857 |
| `rail_oval_right.png` | oval rail the RIGHT cluster straddles | 2.041 |
| `mana_badge.png` | cost tag hanging under a slot | 2.569 |
| `name_plate.png` | name tag under a standard-spell slot | 4.791 |
| `icon_button.png` | chat (left) and settings (right) buttons | 1.000 |
| `medallion.png` | player avatar, top-left (blue ring) | 0.787 |
| `medallion_enemy.png` | opponent avatar, top-right (crimson ring) | 0.789 |
| `bar_frame.png` | empty frame behind an HP / mana bar | 10.000 |
| `timer_banner.png` | clock wings + gem row, top centre | 3.105 |
| `log_panel.png` | combat log, bottom centre | 1.838 |
| `droplet.png` `chat_glyph.png` `gear_glyph.png` | the glyphs the badge and buttons carry | — |

The aspects above are **measured on the delivered art**. Most are mirrored into the constants at
the top of `GameHudScreen` and `SpellSlot`; the two rails and `bar_frame` are the exceptions —
they are stretched to ratios measured on the *reference screenshot* instead, because they are
plain shapes where a few percent of squash is invisible, whereas honouring the PNG's own aspect
moves the rings off the rim they are supposed to straddle.

Batch 1's `plaque.png`, `portrait_frame.png`, `slot.png`, `timer.png` and the short-lived
`rail_arc.png` were deleted along the way.

Keying: `_tools/key_green.py in.png out.png [--square]`. Pass `--square` for anything that must
be 1 : 1; the generator lands within about 1% of circular and the rings have to agree with each
other to the pixel, since the game swaps one for another in place.

## Read the reference before you crop it

Two structural mistakes cost a full round each, and both were misreadings of the screenshot
rather than generation failures:

- The **right cluster is a closed wide oval**, not an arc. Its four small rings *straddle the
  oval's upper rim* — their centres sit above the rail, which passes behind their lower halves —
  and the two standard rings sit inside the lower half, almost touching. The left cluster is the
  opposite: its row sits fully *inside* its oval.
- The **gold lozenges belong to the standard rings, not to the rail.** Each standard ring has
  its own crescent bracket hugging its outer flank with a lozenge at mid-height. Putting them on
  the rail looks nearly right and is wrong.

When something looks like one ornament, brighten the crop and trace the lines before deciding:

```python
from PIL import Image, ImageEnhance
c = Image.open('reference/fight/game.png').crop(box).resize(...)
ImageEnhance.Contrast(ImageEnhance.Brightness(c).enhance(2.5)).enhance(1.8).save(...)
```

The thin gold rails are nearly invisible against the dark arena at 1:1 — that brighten pass is
what revealed both structures.

## Do NOT write prompts for these — hand over the crop

**This is the part that matters.** The first attempt at this batch described each frame in
words and produced a coherent set that was nonetheless *wrong*: heavy antique bronze, thick
bevelled bands, big star finials. The reference is the opposite — thin, pale, cool gold
hairlines on near-black. Prose cannot carry that, and "restrained" / "not busy" does not stop
the model reaching for ornament.

What works: **crop the element out of the reference screenshot, magnify it 3x, attach it, and
ask for that exact thing back with the interior emptied.** The crops live in the scratchpad
recipe at the bottom of this file. The instruction that does the work is a negative one:

> Reproduce THE FRAME IN THE ATTACHED IMAGE EXACTLY - not a frame inspired by it. Match the
> ring thickness, the number of concentric lines, the exact gold tone and how pale it is, the
> dark interior colour, the ornaments and where they sit, the proportions. If your result would
> be thicker, more ornate, more orange, more bronze or more "antique" than the attached crop,
> it is wrong.

Then say what to delete (the spell icon, the numbers, the portrait, the text) and put it on
flat `#00FF6A`. **Keep it short.** Once a crop is attached, naming the fragment to cut out beats
describing how it should look — "cut out only the thin gold arc on the left and the gold diamond
it runs through, delete everything else, put it on flat green" produced an exact result in one
go, where a paragraph of adjectives had been talking the model back into inventing. For the two rails add: *everything the thin gold line does not cover is flat
green, including the area enclosed by it* — they are see-through and the arena shows through.

The locked and unavailable rings are generated from **the model's own gold ring**, re-attached,
so their geometry lines up pixel for pixel with the file they are swapped in for.

Crop boxes used, against the 1672x941 reference:

```python
boxes = {
 'slot_round':      (38,694,196,852),   'slot_round_large':(1200,706,1396,902),
 'name_plate':      (1204,868,1392,914),'mana_badge':      (62,824,162,874),
 'rail_oval':       (20,655,560,935),   'rail_oval_right': (1150,655,1645,941),
 'standard_bracket':(1160,690,1400,925),
 'medallion':       (4,2,158,178),      'medallion_enemy': (1514,2,1668,178),
 'bar_frame':       (142,60,514,110),   'timer_banner':    (690,0,990,95),
 'log_panel':       (600,688,1014,904), 'icon_button':     (14,190,100,276),
}
```

Generated 2026-09-04 in ChatGPT (Volt workspace). Prompt-written first pass (superseded):
https://chatgpt.com/c/6a9aa008-9144-83eb-a9c0-8aa9f09ad58a
Crop-driven second pass (the art actually shipped):
https://chatgpt.com/c/6a9ab239-6530-83eb-9f78-41432725735d

## STYLE-2  (only for an asset with no crop to hand — see the section above first)

```
STYLE: Painted dark-fantasy game UI ornament, matching the attached reference screenshot.
Polished gold and antique brass metalwork, two-tone: dark bronze #6b5326 in the recesses,
bright polished gold #e8d191 on the raised edges, so the metal reads as bevelled and turned,
not as a flat outline. Restrained gothic ornament — thin engraved rails, small pointed
finials, four-pointed star studs. Where the reference shows a gem, it is a small faceted
sapphire #2f7fe0 set in a diamond-shaped brass mount. Interior fields are deep near-black
indigo #0b0916. Slight age and wear on the metal, faint cool rim-light, no bloom.
BACKGROUND: flat SOLID chroma green #00FF6A filling the whole canvas around the object. No
gradient, no vignette, no shadow, no glow, no soft or feathered edges — the ornament must end
in a hard clean edge against the green so it can be keyed out. Nothing else in the image.
VIEW: perfectly flat straight-on 2D orthographic, no perspective, no tilt, no drop shadow.
```

## 5 — `slot_round.png`

```
<STYLE-2>

Draw ONE circular spell-slot ring, centred in a square canvas with a wide green margin.

A heavy polished gold ring, bevelled, with a narrower dark bronze ring nested just inside it,
and a fine engraved bright line on the raised edge of both. Four tiny brass bracket clasps at
the top, bottom, left and right of the outer rim, each no wider than a tenth of the ring.

Measured on the outer edge of the gold and ignoring the green, the ring must be exactly
circular, 1:1.

The inside of the ring is EMPTY dark indigo — the game draws a spell icon into it at runtime.
Do not draw an icon, a symbol, a rune, a number or a gem inside the ring.
```

## 6 — `slot_round_locked.png`

```
<STYLE-2>

Draw ONE circular spell-slot ring for a LOCKED, not-yet-unlocked slot. Same square canvas,
same wide green margin, exactly the same size, thickness and 1:1 proportion as the unlocked
ring — this file is swapped in place of it, so the outer diameter must match.

The metal is tarnished and cold: desaturated grey-bronze, dull, no polished highlight, as if
the gold has gone out. The four bracket clasps are still there but worn.

The inside of the ring is EMPTY dark indigo with ONE small ornament dead centre: a simple
closed padlock rendered in the same dull grey-bronze, no more than a third of the ring's inner
width, flat and unlit. Nothing else inside — no icon, no text, no number, no glow.
```

## 7 — `slot_round_large.png`

```
<STYLE-2>

Draw ONE large circular spell-slot ring for a permanent signature spell, centred in a square
canvas with a wide green margin.

Same family as the smaller ring but grander: a double gold ring — a heavy polished outer band
and a second thinner band inside it, separated by a dark bronze channel. Four four-pointed
gold star studs set into the outer band at the top, bottom, left and right.

Measured on the outer edge of the gold and ignoring the green, the ring must be exactly
circular, 1:1.

The inside is EMPTY dark indigo. Do not draw an icon, a figure, an arrow, a symbol or any
text inside the ring.
```

## 8 — `rail_oval.png`

```
<STYLE-2>

Draw ONE thin ornamental gold rail shaped as a wide flattened OVAL outline, centred with a
green margin. Twice as wide as it is tall, measured on the outer edge of the gold.

It is a hairline rail, not a frame: a single thin polished gold line, doubled to a second
even finer line on the lower half. A small faceted sapphire in a diamond-shaped brass mount
sits at the very top and at the very bottom of the oval, and two smaller plain gold
four-pointed star studs sit on the lower curve, left and right of centre.

The whole inside of the oval is FLAT SOLID CHROMA GREEN, exactly the same green as the
margin — it is a see-through rail, the arena shows through it in game. Do not fill it, do not
tint it, do not draw slots, circles, icons or anything else inside.
```

## 9 — `rail_arc.png`

```
<STYLE-2>

Draw ONE thin ornamental gold rail shaped as a broad open ARC, like a shallow bowl or a
crescent opening upward, centred with a green margin. About one and a half times as wide as
it is tall, measured on the outer edge of the gold.

Same hairline treatment as the oval rail: a single thin polished gold line with a second finer
line following it. A small faceted sapphire in a diamond-shaped brass mount sits at the lowest
point of the arc, and a plain gold four-pointed star stud caps each of the two rising ends.

Everything not covered by the thin gold line is FLAT SOLID CHROMA GREEN, exactly the same
green as the margin. Do not fill the arc, do not draw slots, circles or icons.
```

## 10 — `mana_badge.png`

```
<STYLE-2>

Draw ONE small horizontal tag plate, the kind that hangs under a slot to carry a number.
Centred with a wide green margin. Two and a half times as wide as it is tall, measured on the
outer edge of the metal.

A dark indigo field with a thin polished gold border. Both short ends are cut to a chevron
point that faces outward. Tiny bracket ornaments where the chevrons meet the top and bottom
edges.

The interior is EMPTY dark indigo — the game draws a mana droplet and a number into it. Do not
draw a droplet, a number, a digit or any text.
```

## 11 — `name_plate.png`

```
<STYLE-2>

Draw ONE small horizontal name tag plate, centred with a wide green margin. Four times as wide
as it is tall, measured on the outer edge of the metal.

A dark indigo field with a thin polished gold border, the four corners cut at 45 degrees with
a tiny gold bracket at each cut. Flat and simple — this sits under a spell icon and must not
compete with it.

The interior is EMPTY dark indigo. Do not draw any text, letters or words.
```

## 12 — `icon_button.png`

```
<STYLE-2>

Draw ONE small round button, centred in a square canvas with a wide green margin.

A disc of deep near-black indigo, surrounded by a single thin polished gold ring with two tiny
gold bracket clasps at the left and right of the rim. Slight inner shading so the disc reads
as recessed below the ring.

Measured on the outer edge of the gold, exactly circular, 1:1.

The inside of the disc is EMPTY — the game draws a chat or gear glyph into it. Do not draw a
glyph, a symbol, a letter or an icon.
```

## 13 — `medallion.png`  (player avatar)

```
<STYLE-2>

Draw ONE ornate OVAL portrait medallion frame, centred in the canvas with a wide green margin.
The oval is taller than it is wide, roughly five wide to six tall, measured on the outer edge
of the gold.

A heavy polished gold oval band with a bevelled bronze channel, and inside it a second, glowing
ring of cool sapphire blue light following the same oval. A small brass heraldic shield, with a
blue enamel field and a pale gold sigil, overlaps the bottom of the band. Two tiny gold finials
at the top of the band.

The centre of the oval is EMPTY dark indigo — the game composites the character portrait
inside it. Do not draw a face, a head, a hood, a figure or any silhouette.
```

## 14 — `medallion_enemy.png`  (opponent avatar)

```
<STYLE-2>

Draw the SAME ornate oval portrait medallion frame as the previous image — identical shape,
identical size, identical proportion, identical gold band, identical shield at the bottom —
but with the inner glowing ring in CRIMSON RED #d23b3b instead of sapphire blue, and the
shield's enamel field red instead of blue.

Everything else is unchanged. The centre stays EMPTY dark indigo: no face, no head, no figure.
```

## 15 — `bar_frame.png`

```
<STYLE-2>

Draw ONE long empty horizontal frame for a status bar, centred with a wide green margin. Nine
times as wide as it is tall, measured on the outer edge of the metal.

It is a hairline frame, not a plate: a single thin polished gold line tracing a long shallow
hexagon — both short ends taper to a chevron point, one pointing left and one pointing right.
The interior is dark, almost black.

Leave the interior completely EMPTY — the game fills it with a coloured bar and draws the
numbers over it. Do not draw a fill, a gradient, a bar, numbers or any text.
```

## 16 — `timer_banner.png`

```
<STYLE-2>

Draw ONE wide ornamental clock banner, centred with a wide green margin. Five times as wide as
it is tall, measured on the outer edge of the gold.

Two long thin gold rails sweep down and outward from the top centre like a pair of open wings,
each ending in a pointed finial that curls slightly upward. Where the two wings meet at the
bottom centre they cross over a small horizontal gold bar. Four small four-pointed star studs
are scattered along the rails.

Below the crossing point, in a single row, sit four small faceted diamond-shaped gems in brass
mounts: the left two sapphire blue, the third dark red, the fourth empty dull grey.

The whole area between and above the wings is FLAT SOLID CHROMA GREEN, exactly the same green
as the margin — the clock digits are drawn there by the game. Do not draw any numbers, digits
or a colon.
```

## 17 — `log_panel.png`

```
<STYLE-2>

Draw ONE horizontal panel for a combat log, centred with a wide green margin. Twice as wide as
it is tall, measured on the outer edge of the border.

A rounded rectangle filled with deep near-black indigo at about 85% opacity, bordered by a
single thin polished gold hairline. A small gold bracket ornament inside each of the four
corners. At the top centre of the border, a small gold chevron pointing upward, sitting on the
border line like a collapse handle.

The interior is EMPTY — the game draws the log lines into it. Do not draw any text, letters,
timestamps, rows or separator lines.
```

## 18 — `slot_round_unavailable.png`

The state the player hits constantly: enough spells, not enough mana — or paralysed. It has to
read differently from `slot_round_locked.png`, which means "you do not have this slot yet".
Locked is tarnished bronze with a padlock; unavailable is cold steel with the light cut.

```
<STYLE-2>

Draw the circular spell-slot ring AGAIN — EXACTLY the same outer diameter, ring thickness, four
pointed star finials and 1:1 proportion as the polished gold ring, because this file is swapped
in place of it in the game and the outer edge must line up pixel for pixel.

This is the UNAVAILABLE state, shown when the player cannot cast the spell — out of mana, or
paralysed. The metal is drained and cold: desaturated steel blue-grey, dull and unlit, no warm
gold at all, with a faint frost-like sheen on the raised edges. A thin ring of cold pale-blue
light runs just inside the band, broken into short dashes as if the circuit is cut.

The inside of the ring is EMPTY dark indigo, slightly darker and colder than before. Do NOT draw
a padlock, an icon, a symbol, a cross, a rune, a number or any text inside the ring.
```

## 19 — `droplet.png`, `chat_glyph.png`, `gear_glyph.png`  (one sheet)

Three glyphs in one generation, sliced apart afterwards on the alpha gaps — cheaper than three
round trips and they come out at a consistent weight.

```
<STYLE-2 background rules only>

Draw THREE small glyphs in a single horizontal row, evenly spaced, all the same height, on flat
solid chroma green #00FF6A with a wide margin and clear green gaps between them.

Left: a MANA DROPLET — one teardrop of glowing sapphire blue water with a bright highlight.
Middle: a CHAT BUBBLE — a rounded speech bubble with a small tail at the bottom left and three
dots inside, in solid pale polished gold, flat and unlit.
Right: a SETTINGS GEAR — a simple eight-tooth cogwheel with a round hole in the middle, in the
same solid pale polished gold, flat and unlit.

All three are plain filled shapes with hard clean edges against the green: no frames, no rings,
no plates, no shadows, no glow, no text.
```

## 20 — the two standard spell icons

Not frames, and not on green: these fill the inside of a ring, so they are painted edge to edge
on their own dark field and the slot masks them to a disc at runtime. They land in
`Resources/Spells/<id>/<id>.png`.

```
Forget the green background for this one. Generate ONE square spell icon for <SPELL>, in the
exact style of the round spell icons in the reference screenshot — a glowing arcane sigil
painted on a deep near-black navy field that fills the whole square, edge to edge, with a soft
inner glow.

magic_arrow — a bolt of brilliant electric-blue arcane energy shaped like a barbed arrowhead
streaking diagonally from the lower left toward the upper right, trailing several sharp
splintered light trails, with a few bright sparks.

mirror_ward — a standing humanoid figure seen from the front as a solid dark silhouette,
wrapped in a sphere of brilliant electric-blue arcane light: thin blue energy lines and small
runes surrounding the figure, the light rimming the silhouette.

Cyan-white core, sapphire blue edges, radiating a cool blue glow into the dark field. Straight-on
flat 2D, no perspective, no frame, no ring, no border, no text, no numbers. The dark navy field
must reach all four edges — no green, no white, no transparency, no margin.
```

## Checking the result in-engine

`Game/ScenesV3/Dev/GameHudPreview.tscn` is an offline harness: it fakes a character, an
opponent, a full arsenal and the two standard spells, then fires a few events so the log fills
and the low-mana rings go cold. Play it in the editor and compare against
`reference/fight/game.png` — no server, no matchmaker, no opponent needed.
