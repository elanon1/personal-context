---
type: project
project: Hexbane
area: client
status: active
created: 2026-09-13
updated: 2026-09-13
verified: 2026-09-13
tags: [hexbane, spells, icons, art-direction]
---

# Spell icon art direction

## Direction adopted for the September 2026 redesign

Luminous dark-fantasy magic: one strong magical silhouette on an almost black background, a concentrated bright core, crisp luminous edges, and restrained painterly energy wisps. Use the existing **Magic Arrow** and **Mirror Reflection** images as the visual anchors; the user explicitly requested retaining both. This records the implemented direction, not a claim of separate user approval of every generated image.

Every future spell icon must belong to this family. Do not copy the previous photographic/orb-like legacy assets. Do not add frames or UI decorations to the bitmap: the game supplies those.

## Composition and readability

- Square opaque PNG, approximately 1254×1254 in the current generated set; maintain square aspect ratio. No artificial transparency or baked checkerboards.
- Main silhouette occupies approximately 75–85% of the canvas; leave a dark margin so highlights do not touch the UI frame.
- One dominant motif, a few supporting energy ribbons or fragments. High contrast and clear silhouette matter more than microdetail.
- Background almost black with only a faint local glow; match the references even inside the warm brown interface.
- Light cores may approach ivory/white. Preserve colored edges and dark gaps; avoid an undifferentiated white bloom.
- No words, numbers, labels, watermark, ornate border, environment or multi-panel compositions.
- Inspect beside the reference pair at **48 px** (actual browse/summary icon size) and 160 px. If the spell is recognizable only enlarged, simplify the subject.
- Similar spells must differ by silhouette and motion, not merely brightness or color.

## School palettes

These are visual art families, not a change to combat metadata.

| Family | Dominant energy | Highlight | Shape vocabulary |
|---|---|---|---|
| Arcana | electric blue / cyan | icy white | crystalline edges, precise facets, crescents |
| Ember | orange / amber | hot ivory | tapered flames, forceful descending masses |
| Hex | violet / magenta | pale lavender | fractured seals, bindings, captive cores |
| Venom | acid green / lime | pale yellow-green | viscous droplets, toxic vapor, inward collapse |
| Vitality | emerald / gentle green | warm ivory / gold | ribbons, growth, ascending purification |

## Catalog motifs and runtime locations

| Spell | Runtime PNG folder | Motif |
|---|---|---|
| magic_arrow | magic_arrow | Existing blue diagonal arrow, preserved |
| mirror_reflection | mirror_ward | Existing figure in circular blue ward, preserved |
| firebolt | fireball | A compact blazing orange fire projectile, white-hot pointed core traveling from lower left to upper right, two curling flame trails. Clear teardrop silhouette. |
| heavy_bolt | flamestrike | A massive incandescent orange meteor-like magical hammerhead striking vertically downward, white-hot core, broad descending flame column, small impact fan. Visually heavy, broad silhouette, not a thin arrow. |
| delayed_hex | explosion | A violet cursed seal: bold broken angular diamond surrounding a bright magenta captive core, three separated orbiting arc segments converging inward. Ominous delayed implosion, readable diamond silhouette. |
| poison | poison_dart | A single large acid-green venom droplet with a curved pointed tip, bright viscous rim, sinister dark interior and three small trailing droplets, faint toxic vapor. Simple unmistakable droplet silhouette. |
| paralysis | paralyze | A dark upright forearm and clenched hand restrained by three luminous violet magic bands and taut diagonal magical bindings. Bold restrained fist silhouette, violet-white highlights. |
| cleanse | cure | A brilliant ivory-gold feather-shaped upward sweeping cleansing flame, lifting away and dissolving a few small violet corruption fragments. One clear curved ascending silhouette, soft emerald accents. |
| mend | heal | Two softly glowing emerald ribbons curving inward to stitch together a small central split heart-shaped light, warm ivory center. Compact healing symbol with two clear converging halves. |
| greater_heal | great_heal | A majestic luminous emerald and warm ivory chalice of healing light with three tall crown-like rays rising from its bowl, wide symmetrical silhouette, concentrated golden-white core. |
| regeneration | regeneration | A luminous emerald young sprout with two broad leaves, wrapped by one circular flowing ribbon of green life energy, a few warm ivory motes. Bold living growth silhouette and open circular motion. |
| barrier | arcane_shield | A large upright faceted blue arcane kite shield made entirely of luminous cyan energy, thick crystalline edges and three readable inner facets, central bright vertical ridge. No person or circular dome. |
| dispel | gust | Two sharp opposing cyan-blue crescent blades of magic cutting through and scattering a small violet broken rune at center. Clear asymmetric S-shaped sweeping silhouette with few large shattered fragments. |
| consume_venom | venom_shot | Three acid-green venom droplets pulled inward into a dark star-shaped implosion maw with luminous lime jagged edges, spiraling inward trails. Bold inward collapse silhouette distinct from a single poison droplet. |

Mend and Regeneration now have separate images. Canonical spell id should take priority over a legacy server icon alias; use existing fallback for unknown ids. Keep the old retained folder names for the eleven replaced files to avoid breaking existing references. New Regeneration lives in its own folder.

## Generation recipe for future spells

Use built-in ImageGen, one call per icon. Supply these two local PNGs as **style references only**, not edit targets:

- client:Resources/Spells/magic_arrow/magic_arrow.png
- client:Resources/Spells/mirror_ward/mirror_ward.png

The exact shared production prompt used in this redesign:

```text
Use case: stylized-concept. Asset: ONE square production spell icon for Hexbane dark fantasy game. Match the supplied reference images' high contrast luminous painted energy, near-black background, crisp white-hot edges and restrained wispy magic texture. Reference images are STYLE REFERENCES ONLY, not edit targets. Create an entirely new subject as described. Centered large readable magical silhouette occupying 75-85 percent of image, safe dark margin, readable at 48px. Sophisticated fantasy illustration, coherent shape first, very limited particles, deep nearly-black background throughout. No frame, lettering, captions, numbers, watermark, UI, grid, collage, photorealistic scenery or ornamental border. Square image. Subject: 
<insert the individual subject below>
```

Exact subject prompts:

### firebolt

A compact blazing orange fire projectile, white-hot pointed core traveling from lower left to upper right, two curling flame trails. Clear teardrop silhouette.

### heavy_bolt

A massive incandescent orange meteor-like magical hammerhead striking vertically downward, white-hot core, broad descending flame column, small impact fan. Visually heavy, broad silhouette, not a thin arrow.

### delayed_hex

A violet cursed seal: bold broken angular diamond surrounding a bright magenta captive core, three separated orbiting arc segments converging inward. Ominous delayed implosion, readable diamond silhouette.

### poison

A single large acid-green venom droplet with a curved pointed tip, bright viscous rim, sinister dark interior and three small trailing droplets, faint toxic vapor. Simple unmistakable droplet silhouette.

### paralysis

A dark upright forearm and clenched hand restrained by three luminous violet magic bands and taut diagonal magical bindings. Bold restrained fist silhouette, violet-white highlights.

### cleanse

A brilliant ivory-gold feather-shaped upward sweeping cleansing flame, lifting away and dissolving a few small violet corruption fragments. One clear curved ascending silhouette, soft emerald accents.

### mend

Two softly glowing emerald ribbons curving inward to stitch together a small central split heart-shaped light, warm ivory center. Compact healing symbol with two clear converging halves.

### greater_heal

A majestic luminous emerald and warm ivory chalice of healing light with three tall crown-like rays rising from its bowl, wide symmetrical silhouette, concentrated golden-white core.

### regeneration

A luminous emerald young sprout with two broad leaves, wrapped by one circular flowing ribbon of green life energy, a few warm ivory motes. Bold living growth silhouette and open circular motion.

### barrier

A large upright faceted blue arcane kite shield made entirely of luminous cyan energy, thick crystalline edges and three readable inner facets, central bright vertical ridge. No person or circular dome.

### dispel

Two sharp opposing cyan-blue crescent blades of magic cutting through and scattering a small violet broken rune at center. Clear asymmetric S-shaped sweeping silhouette with few large shattered fragments.

### consume_venom

Three acid-green venom droplets pulled inward into a dark star-shaped implosion maw with luminous lime jagged edges, spiraling inward trails. Bold inward collapse silhouette distinct from a single poison droplet.

For a new spell, describe its mechanical meaning as a single visual metaphor, choose the appropriate family, and compare it against its closest existing neighbor. Never use the same motif for two spells that must be distinguished during combat.

## Integration and verification

- Save the generated final PNG inside the client workspace. Keep original generated sources outside runtime folders.
- Preserve existing Godot import UIDs for replaced PNGs; allow Godot to import new files.
- Update canonical resolution and offline metadata for any new id.
- Review the complete catalog using Godot-rendered 160 px and 48 px previews.
- Evidence from this task: client:verification/spell-icons/ (manifest, build/import logs, preview).
- Full physical-device/in-match visual approval is separate from import, build and catalog-preview verification.

## Source of truth in code

- client:Resources/Spells/
- client:Core/Spells/Spell.cs
- client:Core/Spells/SpellPresentationCatalog.cs
- client:Game/ScenesV3/Dashboard/DashboardScreen.cs
- client:Game/ScenesV3/Lobby/LobbyScreen.cs
- client:verification/spell-icons/manifest.json

