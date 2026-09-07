---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, dev-setup, docker, migrations, testing]
sources: ["server:docs/QUICKSTART-v2.md", "server:CLAUDE.md", "server:AGENTS.md", "server:Makefile", "server:docker-compose.yml", "server:docker-compose.debug.yml", "server:docker-compose.prod.yml", "server:Dockerfile", "server:Dockerfile.debug", "server:local.yml", "server:.env.dist", "server:rebuild.sh", "server:scripts/*", "server:.github/workflows/docker-publish.yml"]
---

# Server developer setup

Verified against the working tree of `hexbane-server` on 2026-09-07. Architecture: [[server-architecture]]. Deployment (Helm, Argo CD, GHCR image): [[infra-and-deploy]].

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| Docker + Docker Compose v2 | any recent | the stack, and the plugin build itself runs inside `heroiclabs/nakama-pluginbuilder:3.27.0` (`Makefile:14`) |
| Go | 1.24.3 (`go.mod:3`; CI uses the same) | `go test`, `go vet`, IDE tooling. A host build cannot be loaded by the container. |
| `migrate` CLI (golang-migrate) | v4.18.3 is what the image ships (`Dockerfile:18`) | `make migrate-*`, schema test. Install with `make install-migrate`. |
| `golangci-lint` | any | `make lint` (no config file in the repo, defaults apply) |
| Node | with built-in `fetch` and `WebSocket` (script comment, `scripts/test_combat_runtime.mjs:2`) | end-to-end combat smoke test |
| `curl`, `jq` | | `scripts/seed_dev_accounts.sh` |

The pluginbuilder tag and the Nakama base image tag must stay identical (`3.27.0` in `Makefile:14`, `Dockerfile:1`, `Dockerfile:13`), otherwise the plugin fails to load with an ABI error.

## First run

```bash
cp .env.dist .env          # required: modules/main.go:32 fatals without a .env next to the binary
make up                    # builds the image (Dockerfile) and starts postgres + nakama
```

`make up` is `docker compose up --build` (`Makefile:127`). The nakama container entrypoint runs, in order (`docker-compose.yml:32-40`):

1. `nakama migrate up` (Nakama's own schema)
2. `migrate -path /nakama/migrations … up` (project migrations baked into the image)
3. `exec nakama --config /nakama/local.yml --session.token_expiry_sec 7200`, with `--google_auth.credentials_json` / `--social.apple.bundle_id` added only when the matching env var is set

Log lines to expect: `Loaded 6 races into registry`, `Spell system module registered successfully`, `Init mage-duel`.

## Daily loop

```bash
make dev        # build plugin in pluginbuilder → docker compose cp into nakama → restart nakama
make test       # go test ./...
make lint       # golangci-lint run ./...
```

`make dev` = `build` + `up -d nakama` + `cp` + `restart` (`Makefile:110-115`). It does **not** rebuild the image, so changes to `data/spells`, `db/migrations`, `local.yml` or `.env` need `make compose-build` / `make up` (or `rebuild.sh`, which additionally copies `local.yml`).

All make targets that exist (`Makefile:31`):

| Target | Does |
|---|---|
| `build` | Linux ELF `build/backend.so` via pluginbuilder, cached in volume `hexbane-go-build-cache` |
| `build-native` | host-OS plugin, tooling only, not loadable in the container |
| `vendor` | `go mod vendor` inside pluginbuilder (vendor/ is committed) |
| `cp`, `dev`, `restart`, `up`, `down`, `clean` | container plumbing |
| `docker-build`, `compose-build`, `compose-up` | image build (`IMAGE ?= hexbane-server:local`) |
| `test`, `lint` | quality |
| `test-balance` | `go run ./cmd/duel-sim -catalog data/spells -seed 42 -matches 1000 -out /tmp/duel-v2.json` |
| `migrate-up`, `migrate-down` (one step), `migrate-create` (interactive, `-seq`), `migrate-force`, `migrate-version`, `install-migrate` | golang-migrate against `DB_URL` |
| `db-reset` | drop + recreate the project DB, rerun both migration sets, restart nakama (`scripts/reset_dev_db.sh`) |
| `db-clear-characters` | delete all characters and reset `account_tutorials`; accounts survive (`scripts/clear_dev_characters.sh`) |
| `db-seed` | create the six per-race test accounts and characters through the public API (`scripts/seed_dev_accounts.sh`) |
| `test-db` | migration round-trip test against a disposable DB (`scripts/test_db_schema.sh`) |
| `init`, `new` | scaffolding |

`make help` mentions `make debug-up`; no such target exists. Use the debug compose file directly (below).

`rebuild.sh` is the pre-Makefile flow: gvm `go1.24.3`, native `go build`, copy plugin and `local.yml`, restart. It only works on a Linux host; on macOS use `make dev`.

## Docker services and ports

| Service | Image | Ports (host) | Notes |
|---|---|---|---|
| `postgres` | `postgres:17.6-alpine` | 5432, 8080 | db `nakama`, user `postgres`, password `localdb`; volume `data`. Port 8080 is a leftover, nothing listens there. |
| `nakama` | built from `Dockerfile` | 7349 gRPC, 7350 HTTP API, 7351 console, bound to `0.0.0.0` so phones on the LAN can connect | volumes `nakama-modules`, `nakama-runtime`, and `./data/spells:/nakama/spells` |

Console: http://localhost:7351 (Nakama default `admin` / `password`). Server key default `defaultkey`. HTTP runtime key `defaulthttpkey` (`local.yml`).

`docker-compose.debug.yml` builds `Dockerfile.debug` (`nakama-dsym:3.27.0`, plugin built with `-N -l`, Delve) and runs Nakama under `dlv --listen=:4000 --headless`; also exposes 2345, Prometheus metrics on 9100 and a Prometheus container on 9090. Start with `docker compose -f docker-compose.debug.yml up --build`.

`docker-compose.prod.yml` runs `ghcr.io/elanon1/hexbane-server:latest` with `restart: always`; the image is pushed by `.github/workflows/docker-publish.yml` on pushes to `main`, tags `v*` / `release-*`, and manual dispatch, after `go test ./...`, `go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim` and `go vet ./...`.

## Environment variables (names only)

| Where | Names | Notes |
|---|---|---|
| `.env` / `.env.dist` (loaded by the plugin) | `PLUGIN_PATH`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_DEFAULT_MODEL`, `OPENAI_TIMEOUT_SECONDS`, `GOOGLE_CREDENTIALS_JSON`, `APPLE_BUNDLE_ID` | the file must exist; no Go code reads the `OPENAI_*` values any more (unverified whether `PLUGIN_PATH` is read). The Docker image copies `.env.dist` to `/nakama/.env` (`Dockerfile:23`). |
| compose → nakama flags | `GOOGLE_CREDENTIALS_JSON`, `APPLE_BUNDLE_ID` | secrets, from host env or `.env`; empty = provider disabled |
| Makefile | `BUILD_DIR`, `PLUGIN_NAME`, `DOCKER_COMPOSE`, `IMAGE_NAME`, `IMAGE_TAG`, `IMAGE`, `PLUGINBUILDER`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_NAME`, `DB_PASSWORD`, `DB_URL` | |
| scripts | `NAKAMA_URL`, `NAKAMA_SERVER_KEY`, `DEV_ACCOUNT_PASSWORD`, `TEST_RACES`, `TEST_DB_URL`, `TEST_DB_CONTAINER`, `TEST_DB_USER`, `MIGRATE_BIN`, `MIGRATIONS_DIR`, `DB_ADMIN_USER` | |

Never put real secret values in docs or commits; `.env` is gitignored.

## Database and migrations

Two migration sets run at every container start: Nakama's own (`nakama migrate up`) and the project's (`golang-migrate`, directory `db/migrations`, baked into the image at `/nakama/migrations`).

Project migrations (working tree; the previous 000001–000016 chain was squashed and is deleted):

| File | Content |
|---|---|
| `000001_initial_schema` | tables `races`, `characters`, `character_spells`, `playstyles`, `playstyle_slots` (`characters.user_id` has a FK to Nakama `users`, magic-point balance CHECK) |
| `000002_reference_data` | the six races (`human`, `elf`, `dark_elf`, `shadow`, `gnome`, `orc`) with stat limits and traits |
| `000003_local_tutorial` | table `account_tutorials` (`training_completed`, `reward_claimed`, `progression_completed`, `reward_receipt`) |

Naming: `make migrate-create` produces `NNNNNN_<name>.up.sql` / `.down.sql` (sequential numbering). Every migration needs a working `down`: `make test-db` applies up, asserts the schema, runs `up` again (no-op), `down -all`, checks Nakama's `users` table survived, then `up` again (`scripts/test_db_schema.sh`). It refuses to run against `nakama`, `postgres` or template databases.

Legacy `spells` table must not exist; the catalog lives only in YAML (see below). Schema details: [[database]].

## Spell catalog mount

Spells are 14 flat YAML files in `data/spells/*.yaml` (no school/tier subfolders any more). The registry walks `/nakama/spells` recursively at boot (`modules/spell_system/registry.go:96`), decodes with `KnownFields(true)`, rejects extra YAML documents, and validates the whole catalog; a bad file fails plugin init. The directory is bind-mounted in `docker-compose.yml:52` and also copied into the image (`Dockerfile:21`), so on the dev stack an edit needs only `docker compose restart nakama`. The `cmd/duel-sim` tool and unit tests read the same files through `-catalog data/spells`.

## Tests

| Command | What it covers |
|---|---|
| `make test` (`go test ./...`) | 34 `_test.go` files: match loop reclaim and rejoin (`engine/core`), protocol handshake (`phase/connecting`), full combat scenarios on the real catalog (`phase/game/*_test.go`, includes a `FuzzCommands` fuzz target), outcome rules (`phase/gameover`), lobby helpers, player state and duel rules, effect handlers, spell registry/identity/version, character limits/starters/tutorial, race traits, playstyles, `cmd/duel-sim` |
| `go test -race ./modules/match/... ./modules/spell_system/... ./cmd/duel-sim` | what CI runs with the race detector |
| `go test ./modules/match/engine/phase/game -run '^$' -fuzz '^FuzzCommands$' -fuzztime=30s` | command-decoder fuzzing (from `docs/spell_system/verification-v2.md`) |
| `make test-balance` / `go run ./cmd/duel-sim -catalog data/spells -seed 42 -matches N [-out file]` | headless duels through the production `GamePhaseState.Advance`, no Nakama or SQL; reports selections, wins by policy / loadout / race, damage, healing, mana, paralyze ticks |
| `NAKAMA_URL=http://127.0.0.1:7350 node scripts/test_combat_runtime.mjs ai\|pvp` | end-to-end against a **seeded** server (`make db-seed` first): authenticates `<race>@test.pl`, joins, auto-drafts, fights, forces a replacement-session reconnect mid-combat, asserts opcode 50 arrives after a final snapshot. `TEST_RACES` picks accounts. Times out after 250 s. |
| `make test-db` (`TEST_DB_URL` required) | migration round trip on a disposable database |
| `scripts/test_db_api.sh` (`TEST_DB_CONTAINER=hexbane-schema-test`, `NAKAMA_URL`) | RPC-level checks for learned spells, magic-point purchase and playstyles against the isolated container; not wired into the Makefile |

Lint: `make lint`. CI also runs `go vet ./...`; `gofmt` is expected on every change (`AGENTS.md`).

## Platform gotchas (verified in repo files)

- **Never copy a host build into the container.** `make build-native` produces Mach-O/PE; Nakama logs `invalid ELF header` (`Makefile:81-83`).
- **`.env` missing = plugin does not start** (`modules/main.go:32-34`).
- **Phones on the LAN**: ports are published on `0.0.0.0` (`docker-compose.yml:53-55`); the client's `network/local_host` must point at this machine's LAN IP (client CLAUDE.md).
- `rebuild.sh` sources gvm and is Linux-only.

## Source of truth in code

- `server:Makefile` — every target, DB URL defaults, pluginbuilder tag
- `server:Dockerfile`, `server:Dockerfile.debug` — image layout (`/nakama/runtime/modules/backend.so`, `/nakama/spells`, `/nakama/migrations`, `/nakama/local.yml`, `/nakama/.env`)
- `server:docker-compose.yml`, `docker-compose.debug.yml`, `docker-compose.prod.yml` — services, ports, entrypoint order
- `server:local.yml` — Nakama runtime config (Go runtime path, matchmaker interval, http key)
- `server:.env.dist` — env variable names
- `server:modules/main.go` — godotenv requirement
- `server:modules/spell_system/{init,registry}.go` — catalog path and validation
- `server:db/migrations/*` — project schema
- `server:scripts/*` — reset, clear, seed, schema test, API test, combat smoke test
- `server:cmd/duel-sim/main.go` — simulator flags
- `server:.github/workflows/docker-publish.yml` — CI test matrix and image publishing
