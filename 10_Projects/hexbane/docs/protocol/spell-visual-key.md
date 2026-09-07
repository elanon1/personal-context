---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, client, visual-key, unimplemented-server]
sources: ["client:docs/opcodes/spell-visual-key.md"]
---

# `visual_key` on Spell (client-only, server not implemented)

Status: **implemented in the client, absent from the server.** The client `Spell` DTO has `[JsonPropertyName("visual_key")]` (`client:Core/Spells/Spell.cs:71-72`), the codec `SpellVisualKey` exists (`client:Game/ScenesV3/Components/SpellVisualKey.cs`), and `RaceSpriteAnimator.BeginCast` consumes it (`client:Game/ScenesV3/Components/RaceSpriteAnimator.cs:194-200`). The server `spell_system.Spell` struct has no such field and no YAML under `server:data/spells/` carries it (grep of the whole server repo: zero hits), so today every spell arrives with `visual_key == null` and the client falls back to its local presets. Treat this page as the intended contract for a future server change.

## Intended transport

Store the string unchanged in the spell catalog and emit it on every `Spell` object: `me.spells` / `me.standard_spells` in [[op_10_game_data]] (and [[op_00_match_entry_data]]), plus spellbook/catalog RPCs. Snapshots and events reference spells by `spell_id`; the client resolves the `Spell` per owner from the opcode-10 loadouts (`DuelProtocol.PresentationSpell`, `client:Application/Match/DuelProtocol.cs:65-74`), so the two players may carry different keys for the same id. No new opcode or command field is needed.

Priority on the client: valid key → local id-based preset → defaults. Malformed/unknown-version keys decode to `null` and fall back silently. The key affects casting presentation only.

## Frozen v1 wire format (verified against `SpellVisualKey.cs`)

43 ASCII chars: `vfx1_` + unpadded Base64URL of exactly 28 bytes. Floats are IEEE-754 binary32 big-endian; negative zero is normalised to zero on encode (`:40`). Decode re-encodes and requires byte-identical output, so non-canonical padding bits, NaN, out-of-range values and unknown flags are rejected (`:62`).

| Offset | Content |
|---|---|
| 0 | animation id 0–10 |
| 1 | effect profile id 0–10 |
| 2 | flags: bit 0 explicit tint (always set), bit 1 trails |
| 3 | ground ring: 1 hidden, 2 visible |
| 4 | intensity float, 0–3 |
| 8 | scale float, 0.5–2 |
| 12, 16, 20, 24 | tint r, g, b, a floats, 0–1 |

Animation ids in order: `attack_1h_01, attack_1h_02, attack_1h_03, attack_2h_01, attack_2h_02, attack_2h_03, attack_2h_04, attack_2h_05, cast_2h, area_2h_01, area_2h_02` (`:12-16`). Profile ids 0–10 are the eleven effect profiles of the dev tool; "automatic" profile/tint/ground-ring choices are resolved to explicit values at encode time (`:30-37`), so automatic and explicit configurations yield the same key.

Example (from the original doc, decodes with the current codec): `vfx1_CgMDAj-AAAA_gAAAP4AAAAAAAAAAAAAAP4AAAA` = `area_2h_02`, profile 3, red tint, intensity 1, scale 1, trails on, ring visible.

## Authoring and verification

- Generator: `client:Game/ScenesV3/Dev/ArenaMaps/ArenaMapsDev.tscn` (F6) — the **VisualKey** field, **Kopiuj klucz** / **Wczytaj klucz** buttons (`ArenaMapsDev.cs:123-154`).
- Checks: `client:Game/ScenesV3/Dev/ArenaMaps/VerifySpellVisualKeys.tscn` — round trip, culture invariance, malformed input, JSON deserialisation into `Spell`, runtime `BeginCast`, dev import, per-player standard-spell resolution.
- Never reorder v1 ids or layout; add a `vfx2_` prefix for incompatible changes.

## Source of truth in code
- `client:Game/ScenesV3/Components/SpellVisualKey.cs` — codec (the only definition of the format)
- `client:Core/Spells/Spell.cs:71` — wire field
- `server:modules/spell_system/spell.go` — `Spell` struct (field absent)
