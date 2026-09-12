---
type: project
project: Hexbane
area: infra
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
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

## Cluster rollout status (2026-09-09)

The user explicitly authorized pushing the server and deploying to their existing cluster, including a full reset of **Hexbane** data if required. Server `main` and `feat/spell-system-redesign` were pushed to `a480c8dfcb77d97a57761f8263de3cf7718ac23f`; remote refs were verified. The merge preserves remote history and keeps the current flat 14-spell catalog; remote edits to retired `mirror_ward` and `restore` YAML remain available in commit `02e509a`.

Local verification passed: `go test ./...`, CI-scoped `go test -race`, `go vet ./...`, and `helm lint helm/hexbane`. GitOps source `/Users/elanon/PycharmProjects/argocd/gitops/argo/apps/hexbane.yaml` matches the server Application example and tracks default-branch `HEAD` with automated sync. It was subsequently updated as described below.

Deployment completed in the cluster after connectivity was restored on 2026-09-09. The old running pod was 152 days old and used digest `a436db6a...`; Argo had already compared server commit `a480c8d` but no chart-template change required a rollout. A preflight pod successfully pulled `sha-a480c8d`, verified Nakama 3.27.0, migrations 1–5 and the 14-spell catalog.

GitOps commit `4ad3d8c` in `elanon1/argocd` sets `spec.sources[0].helm.parameters` → `image.tag: sha-a480c8d` in `gitops/argo/apps/hexbane.yaml`. The source Application was applied to the cluster. The root Application `apps-root` has **no automated sync policy**; changing the GitOps repo alone does not automatically apply its child Application definitions. The child has `automated: {}`, but the controller explicitly skipped this parameter/object update because `selfHeal` is disabled; a one-time Argo sync operation completed the rollout. Do not claim that pushing server code alone automatically deploys new images. Future releases need a published immutable image tag update and application synchronization (or a separately implemented image-promotion automation).

Reset performed only for Hexbane: scaled Deployment to zero, waited for the old pod to terminate, dropped/recreated database `nakama` on its existing dedicated PostgreSQL instance, then synced Argo. Nakama init migrations ran first; application migrations 1–5 completed, `schema_migrations = (5, false)`, six races and zero characters. PVC and PostgreSQL instance retained; old Hexbane accounts/game data removed as authorized.

Verified rollout: Argo `hexbane` **Synced / Healthy**, operation **Succeeded**, source revision `a480c8dfcb77d97a57761f8263de3cf7718ac23f`; pod `hexbane-79c597f85b-r4rr4` ready 1/1 with zero restarts. Image `ghcr.io/elanon1/hexbane-server:sha-a480c8d`, pulled digest `sha256:1304a6758e1b00c220a35ff9d38d46e4805bd9df5daf26fe3d08ae4a9b1ba7ce`. Logs show successful plugin startup, six races and all 14 spells. Through a temporary local service port-forward: `/healthcheck` HTTP 200, `healthcheck` RPC HTTP 200/status ok, `get_entry_spells` RPC HTTP 200/success/count 6/protocol 2/catalog `duel_v2.4`/ruleset `duel_v2`. Preflight pod removed. No gameplay/account-creation smoke test performed.

**Public HTTPS restored 2026-09-09:** the user added DNS-only A records for `hexbane` and `hexbane-console` to `195.42.99.130` in Cloudflare. Public/authoritative DNS resolves both. Cluster recursive DNS still cached NXDOMAIN, so temporarily appended `--acme-http01-solver-nameservers=1.1.1.1:53` to the cert-manager controller for HTTP-01 self-checks. Public challenge endpoints responded correctly. The 32-day-old ACME authorizations/order had expired; removed only failed CertificateRequest `hexbane-tls-3` and triggered immediate renewal with official `cmctl renew -n hexbane hexbane-tls` to bypass failure backoff. New certificate revision 4 is Ready, valid until **2026-12-08T12:58:39Z**, renewal scheduled 2026-11-08. Removed temporary controller flag, verified its rollout and Argo `cert-manager` Synced/Healthy, and deleted diagnostic pod. No persistent cert-manager configuration change.

Public verification with normal hostname resolution and TLS validation: `https://hexbane.elanon.pl/healthcheck` HTTP 200; `healthcheck` and `get_entry_spells` RPCs HTTP 200, six starters, protocol 2/catalog `duel_v2.4`. Console HTTPS returned 200 with valid certificate using explicit address resolution (`curl --resolve`, no TLS bypass); the local macOS resolver still cached NXDOMAIN for console while Cloudflare/public DNS already resolved it. Clients retaining a negative DNS cache may need to wait for expiry. Test-account login and character retrieval were subsequently verified during seeding below; WebSocket/gameplay validation remains separate. No public DNS credentials were handled by the agent.

### Cluster test accounts (2026-09-09)

At the user's request, manually ran the existing seed script against public HTTPS after reset: six race-specific `@test.pl` accounts, default test password, six level-1 characters and 19 starter spell ownership rows. Repeat run authenticated all six and preserved existing characters. The deployment does not automatically seed accounts; PostgreSQL persists them across normal rollouts, but a future database reset must be followed by the seed script. Subsequently, at the user's request, deleted the six seeded characters while retaining all six test accounts/passwords; character creation is now left to the client. Details and credentials: [[database#Test accounts on the cluster (2026-09-09)]].

### Historical local-only context (2026-09-07; superseded by authorization above)

The phrase is not written down in the vault or either repo; this is the observable state. Both repos sit on feature branches (`feat/spell-system-redesign`, `feat/duel-v2-client`) with the whole duel_v2 redesign uncommitted. CI only publishes from `main` and Argo auto-syncs the chart at `HEAD` of the default branch, so **nothing of the redesign has reached the cluster**. The server migration set was rewritten as a "fresh development baseline, not an upgrade" (`docs/spell_system/database-v2.md:3`): the working tree deletes the tracked `000001..000009` + `000012..000016` (28 files; `000010`/`000011` were never committed) and adds `000001_initial_schema`, `000002_reference_data`, `000003_local_tutorial`. Pushing that to `main` would make the `migrate-custom` init container run against a database whose migration history is at version 16 (unverified behaviour of golang-migrate in that case; expect a failed or dirty state and a stuck rollout). Until a prod migration strategy exists, the redesign is local-only by necessity, not just by preference.

## Countdown readiness rollout (2026-09-10)

At the user's explicit request, deployed server commit `b8773ad78ccbbee9211534765f1fe510044036e6` (`fix(match): wait for human arenas before combat countdown`) to the existing cluster. Only loading readiness and its regression tests changed; no database reset or migration changes.

- Local `go test ./...`, CI-scoped `go test -race`, and `go vet ./...` passed. GitHub Actions run `34442050711` completed successfully and published `ghcr.io/elanon1/hexbane-server:sha-b8773ad`.
- GitOps commit `9767ea8acc7380ad34968e9456b70a6aa2e71645` updates `gitops/argo/apps/hexbane.yaml` to `image.tag=sha-b8773ad`. Applied the Application and requested one-time Argo sync, as required by the existing root/self-heal configuration.
- Deployment rollout succeeded. Pod `hexbane-774b5fb78-tbpzg`: ready, zero restarts, digest `sha256:0702e8762688365ba455c5961817859250ab480303e42d7aa0ad0068e85121fb`. Argo `Synced / Healthy`, operation `Succeeded`, source revision `b8773ad78ccbbee9211534765f1fe510044036e6`.
- Startup logs confirm Go plugin initialization, six races, all 14 spells and `Startup done`; no error/fatal entries in inspected startup logs. Public HTTPS `/healthcheck`, RPC `healthcheck` and RPC `get_entry_spells` all HTTP 200; RPC status ok/success.
- No live duel or physical-device test was run in this deployment session. Client source already contains the presentation-ready acknowledgement/buffer fix but installed applications still require an updated build.

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

## Performance rollout (2026-09-10)

Server commit `f64b8fe8306bb60539a5cf974214f8f3b83a8ca5` replaces transient snapshot maps with typed payloads and adds wire/allocation/active/resident benchmarks; see [[2026-09-10-performance]]. CI run `34447986929` passed tests and image publication. GitOps commit `2741f6f` pins `sha-f64b8fe`; the application was applied and explicitly synced as in the preceding rollout.

Verified deployment: pod `hexbane-77c95859cd-zm4hs`, ready, zero restarts; image digest `sha256:0469004f71c2d9147b9ff1a6b84815aabbd51e34705a9ac775ba1c17f4c76677`. Argo reports Synced / Healthy / Succeeded, rollout completed, public `/healthcheck` and RPCs `healthcheck` / `get_entry_spells` returned HTTP 200 and valid JSON. Startup completed with spell registry loaded. No infrastructure sizing or database reset was part of this change. This verifies deployment health, not production concurrent-match capacity.

## Natural fallback deployment (2026-09-12)

User explicitly requested production deployment and confirmed that old clients need not remain supported: enable the15–30s fallback now. Published server commit `b2e7fe2862385d7bd5d756d73c2415804694c1ff` to main; GitHub Actions run `34699124648` passed Go tests/race/vet and image publication.

GitOps commit `99aebe537a42b883c0e6dfcb6fb8d1494c98ec57` pins `image.tag=sha-b2e7fe2` and sets both `nakama.env.HEXBANE_ENABLE_CUSTOM_QUEUE` and `nakama.env.HEXBANE_ENABLE_FALLBACK` to string `true`. Applied the child Application and requested the usual explicit sync. No reset or unrelated infrastructure change.

Preflight DB: migration5 clean,9users/1character,0active character match locks. A complete custom-format pg_dump was saved with mode600 under `/Users/elanon/.codex/backups/hexbane/2026-09-12-before-fallback.dump`:67021bytes, SHA256 `b0b30fb3406b99c0e5f08c0b16d4193689f1950ecf9487372f17b9aaa3f2bed1`; pg_restore list154lines validated archive structure (not a restore drill).

Deployment `hexbane` rolled out successfully. Pod `hexbane-777d4d75b5-tzt8n` ready1/1,0restarts; digest `sha256:bcdda4ae4d1116abeb12ba097498f97b3faddff4b66e402bd8a1c344d90f8928`. Argo `Synced / Healthy / Succeeded`, server revision matches commit. Init logs applied migrations6/7/8; SQL `schema_migrations=(8,false)`. Startup done and public HTTPS healthcheck returned200. Both flags verified from actual Deployment env; authenticated public queue_config confirms enabled.

Runtime verification in progress: dedicated temporary account/character created for one fallback smoke (no existing account modified), opponent assignment after27183ms. Final outcome and cleanup recorded below when complete.

Rollback: set both flags false and repin `sha-f64b8fe` in GitOps, apply/sync after draining games. Migrations6–8 are additive; retain their tables during a runtime rollback. Do not restore backup or drop tables unless a separately diagnosed data failure requires it. Current new normal client can use built-in queue when queue_config returns false; truly old prequeue backend lacks that RPC, so prefer disabling flags on the new backend for a feature-only rollback.

Client source was updated/build-checked locally in the earlier implementation; this deployment does not distribute a new installed client binary. Current backend rejects old built-in normal tickets while custom queue is enabled, as explicitly accepted by the user. Ranked remains built-in/level30.

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
