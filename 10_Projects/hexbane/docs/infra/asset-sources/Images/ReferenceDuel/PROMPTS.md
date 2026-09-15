# Reference duel artwork

Generated with the built-in ImageGen tool on 2026-09-05 from `reference/fight/game.png`.
No CLI/API fallback, Python image editing, or reference screenshot used as the playable HUD.
Godot performs atlas slicing and chroma keying; original RGB outputs remain intact.

## Final assets

- `arena.png`: clean environment, without interface or characters.
- `frames.png`: empty portrait and spell frames, mana badges and standard spell nameplates.
- `ornaments.png`: independent oval rails, top frames, timer, side controls and log panel.
- `duelists.png`: reference characters used **only by the offline visual preview**.
- `sky_mask.png`: source-aligned sky/terrain segmentation for the living atmosphere.
- `hud.png`: intermediate generated full HUD with painted icons, retained as a source deliverable; **not loaded by either scene**.

The production HUD loads spell textures from `Spell.GetTexture()`, local portraits from
`Character.Avatar` / race assets, and the opponent's race avatar from `PrivatePlayerView.RaceId`.
Explicit portrait exports can supply custom portrait textures. No opponent avatar identifier
is present in the current match payload, so the race portrait is the available fallback.

## Prompt set

1. **Environment extraction:** Remove all UI, text, portraits, buttons, spell circles, log panel
   and both standing wizards. Reconstruct the environment behind them. Preserve the purple
   thundercloud vortex at x55%/y25%, gothic castle at x65%/y40%, black spires at x42%/y43%,
   ruined right bridge, cracked platform at y61–70%, dark bottom third, navy/violet palette,
   fine realistic painterly detail and 16:9 framing. No new elements or text.
2. **HUD extraction:** Extract the exact complete HUD, keeping the 1672×941 canvas, positions,
   proportions, fine gold frames, blue/red portrait rims, timer diamonds, chat/gear buttons,
   nine medallions, oval rails and log panel. Remove scenery, standing wizards and all text.
   Empty the red/blue bar fills while keeping their dark tracks. Preserve mana droplets.
   Requested transparency was delivered as RGB checkerboard, so it was not used directly.
3. **Chroma backing correction:** Replace checkerboard and every empty space with flat
   chroma green #00FF00. Preserve every interface pixel, location and size. Keep opaque dark
   tracks, log glass and badges. No text or layout changes. Output `hud.png`.
4. **Empty component frames:** Remove both portrait faces and all nine spell illustrations
   from the generated HUD; replace their circular interiors with the same flat green.
   Preserve thin circular gold frames, coloured rims, crests, badges, nameplates, tracks,
   timer, panel, rails and ornaments in exactly the same positions. Output `frames.png`.
5. **Independent rails:** Remove the nine circular spell frames, mana badges and two spell
   nameplates from the empty HUD. Keep only the two thin oval rails and diamond ornaments;
   reconstruct uninterrupted arcs where spell circles overlapped them. Preserve the rest
   of the atlas. Green backing; no text or additional art. Output `ornaments.png`.
6. **Preview characters:** Extract only the two reference duelists, head to toe, facing each
   other, left blue magic/right magenta magic. Preserve faces, layered leather armour,
   capes, poses and lighting. Two separate cells, equal heights, flat green backing,
   no scenery/UI/text. Output `duelists.png`.
7. **Skyline mask:** Create an exact pixel-aligned semantic segmentation mask of
   `arena.png`, same 1671×941 framing. Pure white for all sky/clouds (including dark
   clouds), pure black for all terrain, mountains, spires, castles, bridges, trees,
   platform and foreground. Trace the skyline including thin peaks. No colours,
   textures or gradients; antialiasing only on contours. Output `sky_mask.png` using
   built-in ImageGen, with no Python image editing.
