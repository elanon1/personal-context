---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, spells, lore]
sources: ["server:docs/spell_system/spell-lore.md", "server:docs/spell_system/spells.md", "server:docs/spell_system/verification-v2.md"]
---

# Spell lore: natures and incantations

Lore metadata describes a spell's identity independently of its mechanics. It lives on every catalog entry as `nature` and `incantation` (`server:modules/spell_system/identity.go:14-17`, inlined into `Spell`) and is validated at load time. Combat rules, names, `school`, costs and the protocol are unaffected. The client currently ignores these fields (display is deferred).

Lore has its own version, `lore_version: 1`, independent of the combat catalog version so editorial changes never reject a combat client (`server:modules/spell_system/identity.go:89-98`).

## Natures (`server:modules/spell_system/identity.go:35-43`)

| ID | Name (as served) | Root word | Primary colour | Accent | Visual style |
|---|---|---|---|---|---|
| `arcana` | Arkana | Ael | `#64B5FF` | `#EAF4FF` | Geometric runes, rings and crystalline surfaces |
| `ember` | Żar | Rath | `#FF713D` | `#A52E25` | Flames, sparks, hot cores and sudden flares |
| `vitality` | Witalność | Ena | `#F0D17A` | `#FFF4D6` | Soft arcs of light, rising motes and flowing waves |
| `venom` | Jad | Vesh | `#A4D84B` | `#506B2B` | Viscous trails, droplets, irregular stains and fumes |
| `hex` | Klątwy | Nyth | `#B58AFF` | `#663D82` | Bindings, seals, fractured signs and inward motion |

Names are served in Polish as written in code. A reflected spell keeps the nature and colour of the original; the mirror flash marks the change of direction.

## Grammar

**action + nature word + optional modifier**, case-sensitive, 2–3 words (`server:modules/spell_system/identity.go:54-85`). The second word must be the root of the declared nature. Dictionary (16 words, `identity.go:45-52`):

| Role | Words |
|---|---|
| action | Tal (send/strike), Or (shield), Vel (reverse/reflect), Ser (restore), Kel (dispel/cleanse), Mor (mark), Dor (bind/stop), Vor (absorb/consume) |
| nature | Ael (pure magic), Rath (ember), Ena (vitality), Vesh (venom), Nyth (hex) |
| modifier | Kar (great/amplified), Lin (lasting/recurring), Sul (delayed/waiting) |

Current assignments (verified against `server:data/spells/*.yaml`):

| Spell | Nature | Incantation |
|---|---|---|
| Magic Arrow | arcana | Tal Ael |
| Mirror Reflection | arcana | Vel Ael |
| Barrier | arcana | Or Ael |
| Dispel | arcana | Kel Ael |
| Firebolt | ember | Tal Rath |
| Heavy Bolt | ember | Tal Rath Kar |
| Mend | vitality | Ser Ena |
| Greater Heal | vitality | Ser Ena Kar |
| Regeneration | vitality | Ser Ena Lin |
| Cleanse | vitality | Kel Ena |
| Poison | venom | Mor Vesh |
| Consume Venom | venom | Vor Vesh |
| Paralysis | hex | Dor Nyth |
| Delayed Hex | hex | Mor Nyth Sul |

## Loader rules

`validateCatalog` rejects an unknown nature, a wrong word count, a nature word that does not match the declared nature's root, a word used in the wrong role, and a full phrase already used by another spell (`server:modules/spell_system/registry.go:172-178`, `identity.go:54-85`). Grammar generates no mechanics; `effects` still defines behaviour.

Adding a spell:

```yaml
nature: ember
incantation: [Tal, Rath, Kar]
```

New words are added to `IncantationWords()` only when a new meaning appears.

## RPC `get_spell_lore`

Registered in `server:modules/spell_system/init.go:9`. No payload, no auth requirement beyond the normal RPC session. Response (`identity.go:90-94`):

```json
{
  "lore_version": 1,
  "natures": [{"id": "arcana", "name": "Arkana", "root": "Ael", "primary_color": "#64B5FF", "accent_color": "#EAF4FF", "visual_style": "..."}],
  "words": [{"word": "Tal", "role": "action", "meaning": "Send or strike"}]
}
```

Definition order is stable. `nature` and `incantation` also appear on spell definitions in `get_entry_spells`, `get_my_spells`, `get_spell`, the spellbook and starter RPCs, character details (`server:modules/character/details.go:116-117`), the tutorial `status` response and full spell definitions inside match messages. See [[rpcs]].

Not implemented: sounds, progressive reveal of phrases, bluffing, hiding spell names. Any future hiding must cover every signal (name, colour, animation, audio).

## Source of truth in code

- `server:modules/spell_system/identity.go` — natures, dictionary, grammar validation, `get_spell_lore`
- `server:modules/spell_system/registry.go` — catalog-level phrase uniqueness
- `server:data/spells/*.yaml` — per-spell `nature` / `incantation`
