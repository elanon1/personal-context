---
type: project
project: Hexbane
area: plans
status: in-progress
created: 2026-09-08
updated: 2026-09-08
verified: 2026-09-08
---

# Remaining spell VFX implementation

Goal: all 14 catalog spells have nature-correct casting and full effects through the same duel, ArenaMapsDev and VfxTest playback. Preserve existing Arrow, Firebolt and Mirror Reflection. No combat/balance changes.

1. Verify current YAML, palette and authoritative events. Eleven missing spells need non-projectile presentations.
2. Add shared actor-bound field lifecycle and distinct geometry/material profiles: Heavy Bolt furnace strike; Hex countdown; Poison fumes; Paralysis bindings; Cleanse sweep; Mend ribbons; Greater Heal chalice; Regeneration motes; Barrier facets; Dispel crescents; Consume Venom implosion.
3. Register scenes, offline cast metadata and presets. Follow target bounds and transforms; keep presets separate from mechanics.
4. Wire persistent statuses by instance id and deadlines. Damage/heal events trigger live pulses and detonation; cleanse/expiry/cancel must never fabricate damage. Utilities show execution sweeps.
5. Build, exercise both saved-spell buttons for all eleven spells and VfxTest tile/swap/clear, test event/snapshot deduplication, removals, reconnect, teardown. GPU captures under verification/spell-fields; inspect frames and iterate.
6. Request focused code review, fix findings, update source notes, decisions and journal. No live Nakama/device validation claims without execution.

## Source of truth in code
- client:Game/FX/SpellField.cs and SpellField.gdshader
- client:Application/Modules/Spell/Effects/SpellEffectManager.Fields.cs
- client:Game/ScenesV3/Dev/ArenaMaps/VerifySpellFields.cs
