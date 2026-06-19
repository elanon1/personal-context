# AGENTS.md — Personal Context Engine Rulebook

> **Read this file FIRST.** It tells you how this vault is organized, where to write,
> what to read before helping, and how things age out. This vault is a second brain
> operated by an AI personal agent (you). You both capture the user's thoughts/decisions
> and retrieve context when helping them create things.

The user: **Filip** — payments/fintech (Volt). Treat this vault as your model of who he is
and how he works, not just a note pile.

---

## Directory map

| Path | Purpose |
|------|---------|
| `_Inbox/` | Raw, unprocessed captures. Fast write now, file later. |
| `_Templates/` | One template per note type. Copy these when creating notes. |
| `00_Identity/` | The living profile = your always-loaded context (who, voice, goals, people). |
| `10_Projects/` | One folder per project (work AND personal). Each has a `_state.md`. |
| `20_Knowledge/` | Atomic evergreen notes, tagged by `domain`. Cross-domain by design. |
| `30_Periodic/` | Daily → Weekly → Monthly → Yearly digests with links to sources. |
| `99_Archive/` | Retired notes & finished projects. Mirrors the original path. |

Numeric prefixes set sort order. Domains are a **frontmatter property, not a folder.**

---

## Frontmatter schema (every note carries this)

```yaml
---
type:     # identity | project | knowledge | person | periodic | inbox
domain:   # personal-life | knowledge | work | projects | creative   (string or list)
status:   # inbox | active | evergreen | archived
created:  YYYY-MM-DD
updated:  YYYY-MM-DD
tags:     []
aliases:  []
---
```

Type-specific extras:
- **project `_state.md`**: `project:`, `repo: <git-url>`, `state: planning|active|paused|done`
- **person**: `relationship:`, `org:`, `comms-style:`, `last-interaction: YYYY-MM-DD`
- **periodic**: `period: daily|weekly|monthly|yearly`, `covers: <range>`
- **archived**: add `archived: YYYY-MM-DD` and `original-path:`

Always set `updated:` when you edit a note. Keep `created:` fixed.

---

## WRITE routing — where a new note goes

1. **Raw thought, no clear home** → `_Inbox/`, `status: inbox`. Don't overthink it.
2. **Decision/info tied to a project** → identify the project:
   - If Claude Code is open inside a git repo, infer the project from the **git remote URL**.
   - Otherwise **ASK the user which project context** before writing. Do not guess.
   - Then append to that project's `10_Projects/<slug>/_state.md` decisions log (dated entry).
3. **Reusable fact / idea / learning** → `20_Knowledge/`, atomic, tagged with `domain`.
4. **Something about the user** (preference, goal, a person) → update the relevant
   `00_Identity/` file **in place**. Never create a duplicate identity note.
5. **Time-bound entry** → the day's `30_Periodic/Daily/YYYY-MM-DD.md` note.

---

## READ order — orient before you help

1. **Always** load `00_Identity/NOW.md` + `00_Identity/profile.md` (who, voice, current focus).
2. If working in a project → that project's `_state.md`.
3. Topical request → search `20_Knowledge/` and the whole vault by tag/keyword (domain-agnostic).
4. Anything involving a person → `00_Identity/People/`.

> If the identity files are still empty, say so and offer to run the onboarding interview
> (see `00_Identity/_onboarding.md`) rather than inventing facts about the user.

---

## Lifecycle / archiving

- Project reaches `state: done` → move its folder to `99_Archive/Projects/<slug>/`,
  set notes `status: archived`, stamp `archived:` and `original-path:`.
- Knowledge note untouched > 12 months and unlinked → **propose** archiving. Never auto-delete.
- Archiving is a `status` transition first, a folder move second. Keep frontmatter intact.

---

## Periodic summaries (on-demand)

Generate only when the user asks.

- **Daily** → summarize the day's `_Inbox/` items, Daily-log entries, and project
  `_state.md` updates into `30_Periodic/Daily/YYYY-MM-DD.md`, with links back to sources.
- **Weekly / Monthly / Yearly** → roll up the next-lower period's summaries. These are
  navigational digests + links, **not** replacements for the source notes.

---

## Conventions

- File/folder slugs: lowercase, hyphenated (`open-banking-sca.md`, `volt-reconciliation/`).
- Link related notes liberally with `[[wikilinks]]`.
- Prefer editing an existing note over creating a near-duplicate.
- When unsure where something belongs, default to `_Inbox/` and flag it for processing.
