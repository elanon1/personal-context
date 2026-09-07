---
type: project
project: Hexbane
area: infra
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, infra, deploy, docker, helm, argocd, ci]
sources: ["vault:10_Projects/hexbane/infra-i-deploy.md", "server:Makefile", "server:docker-compose.yml", "server:docker-compose.debug.yml", "server:docker-compose.prod.yml", "server:Dockerfile", "server:Dockerfile.debug", "server:local.yml", "server:.github/workflows/docker-publish.yml", "server:helm/hexbane/**", "server:deploy/argocd/*", "server:docs/spell_system/database-v2.md", "client:deploy.sh", "client:CLAUDE.md", "client:export_presets.cfg"]
---

# Infra, build and deploy

Verified against the working trees on 2026-09-07. Server repo = `~/GolandProjects/hexbane-server`, client repo = `~/RiderProjects/hexbane`. Variable **names** only; no values.

## Server: local stack (`docker compose`)

| Service | Image / build | Ports (host) | Volumes | Notes |
|---|---|---|---|---|
| `postgres` | `postgres:17.6-alpine` | 5432, 8080 | `data:/var/lib/postgresql/data` | db `nakama`, user `postgres`; healthcheck `pg_isready` |
| `nakama` | `build: .` (Dockerfile) | 7349 gRPC, 7350 HTTP, 7351 console, bound to `0.0.0.0` so a phone on the LAN can reach it | `nakama-modules:/nakama/runtime/modules`, `nakama-runtime:/nakama/runtime`, `./data/spells:/nakama/spells` (bind) | entrypoint: `nakama migrate up` → `migrate -path /nakama/migrations up` → `exec nakama --config /nakama/local.yml --name nakama1 --session.token_expiry_sec 7200` |

- Env passed to the nakama container: `GOOGLE_CREDENTIALS_JSON`, `APPLE_BUNDLE_ID` (both optional, both secrets, taken from host env / `.env`; appended as `--google_auth.credentials_json` / `--social.apple.bundle_id` only when set). The entrypoint deliberately does not use `sh -x` so the Google secret never lands in container logs (`docker-compose.yml:36-45`).
- `local.yml`: Go runtime at `/nakama/runtime/modules`, `http_key` set, `matchmaker.interval_sec 3`, socket + console on `0.0.0.0`.
- Spell catalog: the plugin loads YAML from `/nakama/spells`; locally that is the bind mount of `data/spells/` (14 flat `*.yaml`, no school/tier subfolders), in the image it is `COPY data/spells /nakama/spells`.
- Plugin: `make build` compiles inside `heroiclabs/nakama-pluginbuilder:3.27.0` (must match the Nakama image version, ABI) → `build/backend.so`; `make cp` copies it into the running container; `make dev` = build + `up -d nakama` + cp + restart (fastest loop). `make build-native` produces a host binary that will **not** load in the container (`Makefile:98-104`). `rebuild.sh` is the old Linux/gvm native path.
- Debug: `docker-compose.debug.yml` = `Dockerfile.debug` (nakama-dsym + delve on :4000, `--logger.level DEBUG`, prometheus port 9100) + a `prometheus` service on :9090.
- Prod-style compose: `docker-compose.prod.yml` runs `ghcr.io/elanon1/hexbane-server:latest` with `restart: always` (same entrypoint, no Google/Apple plumbing).
- Other Make targets: `vendor`, `lint`, `test`, `migrate-{up,down,create,force,version}` (golang-migrate, `DB_URL` built from `DB_HOST/DB_PORT/DB_USER/DB_NAME/DB_PASSWORD`), `db-reset` (`scripts/reset_dev_db.sh`, recreates only the project DB), `db-clear-characters`, `db-seed` (`scripts/seed_dev_accounts.sh`: six `<race>@test.pl` accounts through the Nakama API; env `NAKAMA_URL`, `NAKAMA_SERVER_KEY`, `DEV_ACCOUNT_PASSWORD`), `test-db` (`TEST_DB_URL`), `test-balance` (`cmd/duel-sim`).

## Server image (`Dockerfile`)

Stage 1 `nakama-pluginbuilder:3.27.0` → `go mod vendor` → plugin build. Stage 2 `heroiclabs/nakama:3.27.0` + golang-migrate v4.18.3; copies `backend.so`, `data/spells`, `db/migrations`, `local.yml` and **`.env.dist → /nakama/.env`**. The plugin calls `godotenv.Load()` and `log.Fatal`s without a `.env` (`modules/main.go:32-34`), so the copied `.env.dist` is load-bearing. The only variable the Go code reads is `HEXBANE_ENABLE_DEBUG_RPCS` (`modules/character/init.go:17`); the `OPENAI_*` and `PLUGIN_PATH` entries in `.env.dist` are dead.

Image name: `ghcr.io/elanon1/hexbane-server` (tags `latest` on default branch, branch name, git tag, `sha`).

## CI (`.github/workflows/docker-publish.yml`)

Triggers: push to `main`, tags `v*` / `release-*`, `workflow_dispatch`. Jobs: `test` (Go 1.24.3: `go test ./...`, `go test -race` on match/spell_system/duel-sim, `go vet`) → `docker` (buildx, `linux/amd64`, GHA cache, push to ghcr with `GITHUB_TOKEN`). Tests gate the push (the 2026-08-31 note said there were none; that is no longer true).

## Kubernetes: Helm + Argo CD

- Chart `helm/hexbane` 0.1.0, 1 replica, `image.tag latest`, `pullPolicy Always`, pull secret `ghcr-pull-secret` created from `imageCredentials.{registry,username,password}` (values file, committed).
- Init containers (`templates/deployment.yaml:23-48`): `wait-for-db` (busybox `nc -z <db> 5432`) → `migrate-nakama` (`nakama migrate up`) → `migrate-custom` (`migrate -path /nakama/migrations up`). Main container: `nakama --config /nakama/data/local.yml` from a ConfigMap rendered from `nakama.config` (checksum annotation → rollout on change), `--session.token_expiry_sec 7200`.
- Env: `nakama.env` map (`OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_DEFAULT_MODEL`, `OPENAI_TIMEOUT_SECONDS`, all unused by code) + optional `nakama.existingSecret` via `envFrom`. There is **no** `GOOGLE_CREDENTIALS_JSON` / `APPLE_BUNDLE_ID` plumbing in the chart; plain Google ID-token sign-in needs none per the server contract (`docs/client/google-auth.md`), Play Games / Apple would (unverified).
- DB: `database.host hexbane-postgres-postgresql`, `database.name nakama`, `database.password` in values; Bitnami subchart values in `helm/hexbane/values/postgres.yaml` (`bitnamilegacy/postgresql:17.6.0`, 5 Gi PVC).
- Service ClusterIP 7349/7350/7351. Ingress class `traefik`, `cert-manager.io/cluster-issuer: letsencrypt-prod`, TLS on: **`hexbane.elanon.pl`** → 7350 (API), **`hexbane-console.elanon.pl`** → 7351.
- Resources: requests 100m/256Mi, limits 500m/512Mi. Probes: TCP on http port.
- Argo CD `deploy/argocd/application.yaml`: Application `hexbane` in ns `argocd`, **multi-source** (chart from `https://github.com/elanon1/hexbane-server.git` path `helm/hexbane` at `HEAD`, plus `registry-1.docker.io/bitnamicharts/postgresql` 18.0.17 with values `$hexbane/helm/hexbane/values/postgres.yaml`), destination ns `hexbane`, `automated: {}` sync, `CreateNamespace`, `ServerSideApply`. The file is meant to be copied into the GitOps repo `elanonix/argocd` (`gitops/argo/apps/hexbane.yaml`); `repo-secret.yaml` is a placeholder template for a read PAT.

## Prod URLs and client server list

| Client key (`NakamaClientManager.cs:111-134`) | Host | Scheme |
|---|---|---|
| `local` | `NAKAMA_HOST` from `.env`, else `project.godot` `[hexbane] network/local_host` (LAN IP for phone builds), port `NAKAMA_PORT` or 7350 | http |
| `synology` | `192.168.1.9:7350` | http |
| `prod` | `hexbane.elanon.pl:443` | https |

## "Local only" policy (what it means in practice)

The phrase is not written down in the vault or either repo; this is the observable state. Both repos sit on feature branches (`feat/spell-system-redesign`, `feat/duel-v2-client`) with the whole duel_v2 redesign uncommitted. CI only publishes from `main` and Argo auto-syncs the chart at `HEAD` of the default branch, so **nothing of the redesign has reached the cluster**. The server migration set was rewritten as a "fresh development baseline, not an upgrade" (`docs/spell_system/database-v2.md:3`): the working tree deletes the tracked `000001..000016` and adds `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial`. Pushing that to `main` would make the `migrate-custom` init container run against a database whose migration history is at version 16 (unverified behaviour of golang-migrate in that case; expect a failed or dirty state and a stuck rollout). Until a prod migration strategy exists, the redesign is local-only by necessity, not just by preference.

## Client build and deploy

- Godot 4.5 mono, `Godot.NET.Sdk/4.5.2`, `net9.0` (`hexbane.csproj:1-3`). `<StartProgram>` still points at a Windows path (`hexbane.csproj:5`), harmless on macOS.
- `deploy.sh` (rewritten for this Mac): Godot at `/Applications/Godot.app/Contents/MacOS/Godot` (`GODOT=`), ADB at `~/Library/Android/sdk/platform-tools/adb` (`ADB=`), refuses without an authorized device, reads the baked host from `project.godot` and warns if `http://<host>:7350/` does not answer, then `--headless --export-debug "Android" hexbane1.apk` and `adb install -r`. Budget 10 minutes; APK is large.
- Export presets: Windows Desktop (`../export/test.exe`), Android (`./hexbane1.apk`, package `pl.elanon.hexbane`), macOS (`../export/test_mac.app`), iOS (no path). Desktop presets exclude the SD race sprites, mobile presets exclude the HD ones (`export_presets.cfg:11,82,308,566`), see [[assets-pipeline]].
- `res://.env` is never packed into an export (Godot ignores dotfiles), so exported builds fall back to `project.godot` `[hexbane]` values: `network/local_host`, `graphics/race_sprites`, `auth/google_client_id`, `auth/google_client_secret`, `auth/google_loopback_port`.
- Nakama on the dev box must listen on `0.0.0.0:7349-7351` for the phone (it does, see compose above).

## Secrets: where they are (names only)

- Server `.env.dist` (tracked): contains an `OPENAI_API_KEY` with a real-looking value; unused by code. `GOOGLE_CREDENTIALS_JSON` / `APPLE_BUNDLE_ID` are empty placeholders there and documented to live only in the gitignored `.env`.
- Server `helm/hexbane/values.yaml:15` (tracked): `imageCredentials.password` holds a real-looking GitHub PAT; `database.password` is a placeholder.
- Client `.env` (tracked): `AUTH_EMAIL`/`AUTH_PASSWORD` dev login and `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` (Desktop-app OAuth client; that secret is designed to ship, see `docs/client/social-sign-in.md`).
- Client `project.godot:70-71` (staged, not at HEAD): the same Google Desktop-app id/secret; `client_secret_*.json` in the repo root is untracked but listed as `<Content>` in `hexbane.csproj:31`.

None of these have been rotated; see [[repos-and-branches]] and the vault-state audit.

## Source of truth in code
- `server:Makefile` — build/dev/migrate/db-* targets, pluginbuilder version, `DB_URL` composition
- `server:docker-compose.yml`, `server:docker-compose.debug.yml`, `server:docker-compose.prod.yml` — local stacks, ports, mounts, entrypoint order
- `server:Dockerfile`, `server:Dockerfile.debug` — image layout, `.env.dist` copy, migrate binary
- `server:local.yml` — Nakama runtime config (also embedded in `helm/hexbane/values.yaml` `nakama.config`)
- `server:.github/workflows/docker-publish.yml` — CI triggers, tests, image tags
- `server:helm/hexbane/templates/*.yaml`, `server:helm/hexbane/values.yaml`, `server:helm/hexbane/values/postgres.yaml` — k8s deployment, ingress hosts, init containers
- `server:deploy/argocd/application.yaml` — Argo CD Application (multi-source, automated)
- `server:modules/main.go:32` — `godotenv.Load()` hard requirement; `server:modules/character/init.go:17` — `HEXBANE_ENABLE_DEBUG_RPCS`
- `client:deploy.sh`, `client:export_presets.cfg`, `client:project.godot` `[hexbane]`, `client:Application/Nakama/NakamaClientManager.cs:106-160` — client build, presets, server table, `LocalHost()`
