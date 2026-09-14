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
Scope: a local dark-fantasy promotional website with a cinematic hero, interactive CSS 3D sigil, spell presentation, six-race explorer, and Android/iOS download previews. Real store listings and publishing remain pending.

## Architecture

React + TypeScript using the Sites Vinext starter and its Next-compatible App Router, Vite, Tailwind CSS, and Cloudflare build support. Retain the starter lockfile and portable execution profile. This choice reuses the available Sites workflow; it is not a requirement imposed by the Godot client or Go backend. The starter includes optional UI, auth and database utilities; none is connected to the promotional page. Vinext is pinned to a beta release in the supplied starter.

- `app/`: route composition, root metadata/layout, and global layout styles.
- `components/sections/`: Hero, Gameplay, Races, and Download sections; client download panel switches platform and QR together.
- `components/effects/arcane-scene.tsx`: pointer-responsive CSS 3D rings, particles, click-triggered light pulse, and motion pause control.
- `components/site/`: shared site navigation.
- `content/game.ts`: curated public-facing game content, with source provenance.
- `config/site.ts`: navigation, language, and platform destinations with explicit `placeholder` flags.
- `lib/download-qr.ts`: server-generated SVG QR data URLs using `qrcode`; no external QR service.
- `styles/tokens.css`: emerald/old-gold palette and local Cinzel, Cormorant Garamond, and Inter fonts. Font licenses are distributed in `public/fonts/`.
- `public/images/`: WebP exports of existing game loading/arena art, six portraits, and three spell icons; all image files total approximately 1035 KiB. `public/videos/` remains reserved for real gameplay footage.
- `public/favicon.svg`: simple initial H monogram, not final game branding.
- `components/ui/`, `hooks/`, `lib/`, `vendor/`: supplied starter utilities.
- `build/`, `scripts/`, `.openai/hosting.json`, `vite.config.ts`: starter build/runtime infrastructure.
- `db/`, `drizzle/`, `drizzle.config.ts`, `app/chatgpt-auth.ts`: optional starter support; no database/storage bindings or login route enabled.

The website is independent of Nakama. Rendering does not query either game repository or backend. Curated content avoids runtime coupling and accidental publication of internal data. The user explicitly requested temporary store destinations: Google Play uses `com.example.hexbane`, App Store uses `id0000000000`. Both QR codes encode those exact placeholder URLs. Change `platforms` in `config/site.ts` to replace them. The page labels links and QR as previews; no downloadable binary is supplied. There is no invented trailer or community link. Do not present unconfirmed store availability, ratings, seasons, or release dates as facts.

## Local commands

Node >=22.13.0. Dependencies are installed; on a new checkout run `npm ci`.

- `npm run dev`: local development server (use the URL printed at startup).
- `npm run typecheck`: TypeScript validation.
- `npm run build`: production build.
- `npm start`: local preview of built Cloudflare Worker output.
- `npm run lint`: available ESLint check.

Build and TypeScript checks passed on 2026-09-14. No browser visual QA or deployment was performed. Existing `.idea/` files were preserved and ignored. No game/server code was changed. Website documentation remains in this vault; repository README only points here.

## Next work

Replace the two placeholder listings with real destinations and revise release-state copy, run responsive browser and physical-phone scan checks, then prepare hosting when requested. Add actual gameplay footage only when available; the current arena image is game artwork rather than a gameplay screenshot.

## Source of truth in code

- `website:app/{page,layout}.tsx`, `content/game.ts`, `config/site.ts`, `styles/tokens.css`, `package.json`
- `client:Core/Characters/RaceCatalog.cs`, `Core/Spells/StandardSpells.cs`
- `server:modules/spell_system/standard.go`, `data/spells/mirror_reflection.yaml`
- Related contracts: [[progression]], [[design-system]].

## Promotional redesign — 2026-09-14

The user authorized a designer presentation with 3D-style effects and placeholder download links. The visual direction combines existing emerald sanctuary art, warm gold display typography, fine rules, and dark editorial sections. All visible marketing copy remains English, consistent with the original site.

The hero uses CSS perspective and rotateX/rotateY/translateZ, animated orbit rings and particles. Pointer tilt is limited to mouse input; the light burst has a keyboard-accessible button and a screen-reader status. A pause button stops continuous motion. `prefers-reduced-motion` disables animation and smooth scrolling. This is CSS 3D, not a WebGL scene or playable combat simulation.

Race and platform selectors use the supplied Radix-based Tabs components for keyboard navigation and selected-state semantics. QR codes have a four-module quiet zone, medium error correction, and black-on-white modules. Generation follows [node-qrcode documentation](https://github.com/soldair/node-qrcode). No third-party QR endpoint receives URLs. Local image conversion uses Sharp; source game assets are untouched. The download flow renders on the server and receives two precomputed QR images.

Validation: production build and TypeScript passed after the redesign; local homepage responded HTTP 200; static font/image references resolved. No browser visual inspection or physical phone testing was performed in this task.

Additional verification: both generated QR images were decoded by Apple Vision on macOS to their exact configured Google Play/App Store URLs. This verifies machine-readable payloads, not physical-phone scanning.
