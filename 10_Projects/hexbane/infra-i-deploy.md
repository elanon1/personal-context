---
type: project
project: Hexbane
domain: [projects, work]
status: active
created: 2026-08-31
updated: 2026-08-31
tags: [hexbane, kubernetes, helm, argocd, docker, nakama, postgres, ci]
aliases: [hexbane-infra]
---

# Hexbane — infra, build i deploy

Stan na 2026-08-31. Część [[10_Projects/hexbane/_state|Hexbane]]. Klaster ten sam co w [[10_Projects/cluster-agent/_state|Cluster Agent]] (GitOps `elanonix/argocd`).

## Serwer — build lokalny

- `make build` — kompilacja pluginu **w kontenerze** `heroiclabs/nakama-pluginbuilder:3.27.0` → `build/backend.so` (musi zgadzać się z wersją Nakamy — ABI). `make dev` = build + `docker compose up -d nakama` + cp `.so` + restart (najszybsza pętla). `make vendor`, `make lint`, `make test`, `make migrate-{up,down,create,force,version}` (golang-migrate, `DB_URL` domyślnie `localhost:5432/nakama`, `postgres/localdb`).
- `rebuild.sh` — build natywny (gvm go1.24.3) + cp + restart; tylko Linux.
- `docker-compose.yml`: `postgres:17.6-alpine` (`localdb`, port 5432) + `nakama` z `Dockerfile` (porty 7349 gRPC / 7350 HTTP / 7351 console; entrypoint `nakama migrate up` → run z `local.yml`; bind `./data/spells:/nakama/spells`). `docker-compose.debug.yml`: `Dockerfile.debug` (nakama-dsym + delve :4000) + Prometheus. `docker-compose.prod.yml`: `ghcr.io/elanon1/hexbane-server:latest`, `restart: always`.
- `local.yml`: runtime Go, `http_key defaulthttpkey`, `matchmaker.interval_sec 3`, socket/console `0.0.0.0`. Konsola Nakamy dev: `admin/password`.
- `Dockerfile`: pluginbuilder → `heroiclabs/nakama:3.27.0` + golang-migrate 4.18.3; kopiuje `backend.so`, `data/spells → /nakama/spells`, `db/migrations → /nakama/migrations`, `local.yml`, **`.env.dist → /nakama/.env`** (plugin robi `godotenv.Load()` i pada bez `.env`).

## CI/CD

- `.github/workflows/docker-publish.yml`: push na `main`, tagi `v*`/`release-*`, ręcznie → `ghcr.io/elanon1/hexbane-server` (`latest`, branch, tag, `sha`), `linux/amd64`, cache GHA. **Bez testów/lintu przed pushem.**
- Helm `helm/hexbane` (chart 0.1.0): 1 replika, `image.tag latest` + `pullPolicy Always`; pull-secret `ghcr-pull-secret`; initContainers `wait-for-db` → `migrate-nakama` (`nakama migrate up`) → `migrate-custom` (`migrate` na `/nakama/migrations`); ConfigMap `local.yml` z checksumem (rollout przy zmianie); Service ClusterIP; Ingress traefik + `letsencrypt-prod`: **`hexbane.elanon.pl`** (API, 7350) i **`hexbane-console.elanon.pl`** (7351); zasoby 100m/256Mi → 500m/512Mi; env `OPENAI_*` (puste, nieużywane przez kod) + opcjonalny `existingSecret`. Baza: `hexbane-postgres-postgresql`, hasło z values (`changeme`).
- `helm/hexbane/values/postgres.yaml`: subchart Bitnami (`bitnamilegacy/postgresql:17.6.0`, 5 Gi).
- Argo CD `deploy/argocd/application.yaml`: Application `hexbane` (ns `argocd`), **multi-source** — chart z repo (`ref: hexbane`, `helm/hexbane`) + `bitnamicharts/postgresql 18.0.17` z values `$hexbane/helm/hexbane/values/postgres.yaml`; destination ns `hexbane`; `automated` sync, `CreateNamespace`, `ServerSideApply`. Plik do skopiowania do GitOps (`gitops/argo/apps/hexbane.yaml`); `repo-secret.yaml` z placeholderem PAT.
- Klient wskazuje prod `hexbane.elanon.pl:443` (https) i `synology 192.168.1.9:7350` (devlog 30.12.2025: „server on Synology”).

## Sekrety i env — fakty

- **Zacommitowane na twardo**: `hexbane-server/.env.dist:4` (klucz OpenAI `sk-proj-…`), `helm/hexbane/values.yaml:15` (PAT `ghp_…` do ghcr), klient `hexbane/.env` (dev e-mail/hasło; csproj pakuje `.env` do builda). Odnotowane, **nie rotowane**.
- Jedyna zmienna czytana przez Go: `HEXBANE_ENABLE_DEBUG_RPCS`. Baza przekazywana flagą `--database.address`.

## Klient — build/deploy

- Godot 4.5 mono; C# hot reload; `export_presets.cfg`: Android (`com.example.$genname`, tylko `internet` + `access_network_state`, wszystkie ABI), Windows, macOS (`hexbane.game`), iOS (bez bundle id). Ścieżki eksportu poza repo (`../export/`).
- `deploy.sh` (APK + WiFi ADB) i `hexbane.csproj` `StartProgram` = ścieżki Windows/WSL2 — na tym Macu brak działającej ścieżki deployu na telefon.
