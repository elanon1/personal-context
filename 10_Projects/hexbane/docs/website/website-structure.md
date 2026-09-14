---
type: project
project: Hexbane
area: website
status: active
created: 2026-09-14
updated: 2026-09-14
verified: 2026-09-14
tags: [hexbane, website, setup]
sources: ["website:package.json", "website:app/page.tsx", "client:Core/Characters/RaceCatalog.cs", "server:data/spells/mirror_reflection.yaml"]
---

# Website project structure

Local project: `/Users/elanon/GolandProjects/hexbane-website`.
Scope: a runnable promotional website foundation, with preliminary English copy. Final art direction, media, store links, and publishing remain future work.

## Architecture

React + TypeScript using the Sites Vinext starter and its Next-compatible App Router, Vite, Tailwind CSS, and Cloudflare build support. Retain the starter lockfile and portable execution profile. This choice reuses the available Sites workflow; it is not a requirement imposed by the Godot client or Go backend. The starter includes optional UI, auth and database utilities; none is connected to the promotional page. Vinext is pinned to a beta release in the supplied starter.

- `app/`: route composition, root metadata/layout, and global layout styles.
- `components/sections/`: Hero, Gameplay, and Races homepage sections.
- `components/site/`: shared site navigation.
- `content/game.ts`: curated public-facing game content, with source provenance.
- `config/site.ts`: navigation, language, and nullable public destinations.
- `styles/tokens.css`: colors and font roles, initially using system fonts.
- `public/images/`, `public/videos/`: future optimized promotional assets.
- `public/favicon.svg`: simple initial H monogram, not final game branding.
- `components/ui/`, `hooks/`, `lib/`, `vendor/`: supplied starter utilities.
- `build/`, `scripts/`, `.openai/hosting.json`, `vite.config.ts`: starter build/runtime infrastructure.
- `db/`, `drizzle/`, `drizzle.config.ts`, `app/chatgpt-auth.ts`: optional starter support; no database/storage bindings or login route enabled.

The website is independent of Nakama. Rendering does not query either game repository or backend. Curated content avoids runtime coupling and accidental publication of internal data. Download/community/trailer URLs remain null until public destinations are confirmed. Do not present unconfirmed store availability, ratings, seasons, or release dates as facts.

## Local commands

Node >=22.13.0. Dependencies are installed; on a new checkout run `npm ci`.

- `npm run dev`: local development server (use the URL printed at startup).
- `npm run typecheck`: TypeScript validation.
- `npm run build`: production build.
- `npm start`: local preview of built Cloudflare Worker output.
- `npm run lint`: available ESLint check.

Build and TypeScript checks passed on 2026-09-14. No browser visual QA or deployment was performed. Existing `.idea/` files were preserved and ignored. No game/server code was changed. Website documentation remains in this vault; repository README only points here.

## Next work

Choose the primary public call to action, supply approved gameplay screenshots/trailer and store/community links, refine final visual design, then validate responsive layouts and prepare hosting.

## Source of truth in code

- `website:app/{page,layout}.tsx`, `content/game.ts`, `config/site.ts`, `styles/tokens.css`, `package.json`
- `client:Core/Characters/RaceCatalog.cs`, `Core/Spells/StandardSpells.cs`
- `server:modules/spell_system/standard.go`, `data/spells/mirror_reflection.yaml`
- Related contracts: [[progression]], [[design-system]].
