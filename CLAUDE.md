# CLAUDE.md — Personal Context Engine

Compact operating guide. This vault is a **second brain you (the AI) run for Filip**
(payments/fintech, Volt): capture his thoughts/decisions in, read context back out when
creating. Treat it as your model of who he is and how he works.

## Map
| Path | Holds |
|------|-------|
| `_Inbox/` | Raw captures, filed later |
| `_Templates/` | One template per note type |
| `00_Identity/` | Living profile: `NOW`, `profile`, `goals`, `tools-stack`, `People/` |
| `10_Projects/<slug>/_state.md` | One folder per project (work + personal) |
| `20_Knowledge/` | Atomic evergreen notes, tagged by `domain` |
| `30_Periodic/` | `Daily/ Weekly/ Monthly/ Yearly` digests |
| `99_Archive/` | Retired notes & finished projects |

Domains are a **frontmatter property, not a folder.** Prefixes set sort order.

## READ first — orient before helping
1. **Always** `00_Identity/NOW.md` + `profile.md` (who, voice, current focus).
2. In a project? → that project's `_state.md`.
3. Topical? → search `20_Knowledge/` + whole vault by tag/keyword (domain-agnostic).
4. About a person? → `00_Identity/People/`.
> Identity files empty → offer onboarding (`00_Identity/_onboarding.md`); never invent facts.

## WRITE routing — where a capture goes
1. Raw / unsure → `_Inbox/` (`status: inbox`).
2. **Project decision** → find the project: match the **git remote URL** if open in a repo,
   else **ASK which project** (don't guess) → append a dated entry to its `_state.md`.
3. Reusable fact / idea → `20_Knowledge/` (atomic, `domain`-tagged).
4. About Filip (pref / goal / person) → edit the `00_Identity/` file **in place** (no duplicates).
5. Time-bound → `30_Periodic/Daily/YYYY-MM-DD.md`.

## Frontmatter (every note)
```yaml
type: identity|project|knowledge|person|periodic|inbox
domain: personal-life|knowledge|work|projects|creative   # string or list
status: inbox|active|evergreen|archived
created: YYYY-MM-DD
updated: YYYY-MM-DD      # bump on every edit; keep `created` fixed
tags: []
aliases: []
```
Extras — project: `project, repo, state(planning|active|paused|done)` · person: `relationship, org, comms-style, last-interaction` · periodic: `period, covers` · archived: `archived, original-path`.

## Wiki / linking
- The vault is a **wiki, not siloed folders** — interlink liberally with `[[wikilinks]]`
  (a `[[name]]` that doesn't exist yet is a fine forward-reference).
- Slugs: lowercase-hyphenated (`open-banking-sca`, `volt-reconciliation/`).
- Prefer editing an existing note over creating a near-duplicate.

## Lifecycle
- Project `state: done` → move folder to `99_Archive/Projects/<slug>/`, set `status: archived`,
  stamp `archived:` + `original-path:`.
- Knowledge note stale >12mo & unlinked → **propose** archiving; never auto-delete.

## Summaries (on-demand only)
Daily rolls up `_Inbox/` + the day's log + project `_state.md` changes →
`30_Periodic/Daily/`. Weekly/Monthly/Yearly roll up the level below. Always digests **+ links**,
not rewrites.

---
_Conventions in full: this file is canonical. `README.md` orients humans; `_Templates/` shows each note type._
