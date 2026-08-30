---
type: project
project: Cluster Agent
domain: projects
status: active
state: active
repo: https://github.com/elanonix/argocd
created: 2026-07-28
updated: 2026-07-28
tags: [agents, n8n, kubernetes, argocd, homelab, security]
aliases: [cluster-agent, agent-klastra]
---

# Cluster Agent — State

## Summary

Sub-agent n8n, który diagnozuje problemy w klastrze Kubernetes na żywo („czemu obsidian nie wstaje?") i stawia nowe rzeczy („dodaj mi X"), rozmawiając przez istniejącego bota Telegram za `Personal Assistant (Master)`. Diagnoza dzieje się bezpośrednio na klastrze przez MCP; **każda trwała zmiana idzie przez pull request** w `elanonix/argocd`, który Filip merguje ręcznie.

Kluczowa właściwość konstrukcyjna: **granicą uprawnień jest RBAC ServiceAccountu, nie prompt i nie flagi serwera MCP.** API server to jedyne miejsce, którego model nie obejdzie. Wszystko inne to co najwyżej drugi pas bezpieczeństwa.

Projekt niezależny od [[10_Projects/agent-workforce/_state|Agent Workforce]], choć dzieli z nim repo, mastera i filozofię kolejki zatwierdzeń przez PR-y. Tamten robi z agentów pracowników pod side income; ten jest narzędziem operacyjnym do własnej infrastruktury.

## Status

`active`. Warstwa klastrowa **stoi i działa**; warstwa n8n czeka.

**Zrobione:**
- Chart `gitops/apps/mcp-server` rozszerzony o opcjonalny ServiceAccount, dwupoziomowy RBAC i NetworkPolicy — wszystko domyślnie wyłączone, z testem regresji pilnującym, że `mcp-actual` renderuje się bajt w bajt tak samo.
- `mcp-k8s` żyje w namespace `mcp` (obraz `ghcr.io/containers/kubernetes-mcp-server`, natywny Streamable HTTP, **bez mcp-proxy** — świadomie, bo to ta warstwa potrafiła się zawieszać przy Obsidianie). 19 narzędzi.
- Uprawnienia: czytanie całego klastra **bez `secrets` i bez `pods/exec`**, jedna mutacja (`pods: delete`) w 11 namespace'ach aplikacyjnych. `kube-system` poza zasięgiem.
- Kontrakt uprawnień jest wykonywalny: `gitops/mcp/k8s/rbac-check.sh`, 25 asercji, przechodzi 25/25.

**Zablokowane:** Taski n8n (`Propose Change`, `Cluster Agent`, wpięcie do mastera, poranny raport) — token MCP do n8n wygasł w trakcie sesji i serwer odrzuca nagłówek Authorization (HTTP 401). Nic dalej nie ruszy bez reautoryzacji.

Spec i plan wdrożenia leżą w repo: `docs/superpowers/specs/2026-07-27-n8n-cluster-agent-design.md` oraz `docs/superpowers/plans/2026-07-27-cluster-agent.md`.

## Decisions log

- **2026-07-27 — Read-only + wąska lista akcji operacyjnych; trwałe zmiany wyłącznie przez git.** Agent czyta klaster bezpośrednio, ale cokolwiek chce zmienić na stałe, commituje jako PR. **Why:** ArgoCD ma `syncPolicy.automated`, więc ręczna mutacja albo zostanie nadpisana, albo zacznie produkować dryf; PR daje audyt w historii commitów i rollback przez revert.

- **2026-07-27 — Gotowy serwer MCP zamiast własnego, a granicą jest RBAC.** Świadomie **nie** ustawiono `--read-only` ani `--disable-destructive`. **Why:** żadna z tych flag nie potrafi wyrazić „czytaj wszystko, ale mutuj jedną rzecz", a ich obecność sugerowałaby granicę w miejscu, które granicą nie jest. MCP nadal *ogłasza* `pods_exec`, `pods_run` czy `resources_delete` — agent po prostu dostaje na nich `Forbidden`, i prompt uczy go, że to stan zamierzony, a nie awaria do obejścia.

- **2026-07-27 — Restart = skasowanie poda, nie `patch` na Deploymencie.** Pierwotny pomysł zakładał adnotację `restartedAt`. **Why:** patch tworzy dryf względem gita i walczyłby z `selfHeal`, a `patch` na Deploymencie z natury pozwala też podmienić obraz i liczbę replik — RBAC nie umie zawęzić go do samej adnotacji. Kasowanie poda daje ten sam efekt, jest bezdryfowe i wymaga radykalnie węższego uprawnienia.

- **2026-07-27 — Bez dostępu do `secrets` i bez `pods/exec`, żadnym czasownikiem.** **Why:** agent raportuje na Telegram, więc czytanie sekretów czyni z niego kanał wycieku; exec to dowolny kod w kontenerze, czyli obejście wszystkich pozostałych ograniczeń naraz. Do diagnozy „sekret się nie zsynchronizował" wystarczy status `ExternalSecret`. Do rozważenia później, świadomie, jeśli diagnostyka bez execa okaże się za słaba.

- **2026-07-27 — `propose_change` jest osobnym mikro-workflow bez LLM.** Agent odpowiada za treść manifestu, nie za hydraulikę gita (branch → commit → PR). **Why:** plumbing, który nie przechodzi przez model, nie może zostać przez model pomylony. Ma też własną, twardą walidację: odrzuca każdą ścieżkę spoza `gitops/`, więc nawet agent zmanipulowany promptem nie zapisze do `.github/workflows/`.

- **2026-07-27 — Poranny raport nie idzie przez mastera.** Osobny workflow woła sub-agenta i pisze wprost na Telegram. **Why:** żeby codzienny raport nie zaśmiecał pamięci rozmowy. Cena: raport nie trafia do historii, więc pytanie „a co z tym podem?" nie ma kontekstu — stąd wymóg, żeby raport wymieniał konkretne nazwy zasobów, nigdy „ten problematyczny pod".

- **2026-07-28 — COFNIĘTE uprawnienie `patch` na Argo Application.** Recenzja bezpieczeństwa wykazała, że składa się ono w cluster-admin: AppProject `default` ma `sourceRepos`, `destinations` i `clusterResourceWhitelist` ustawione na `*`, a `argocd-application-controller` ma `*/*/*`. Jeden patch wstawiający inline `spec.sources[].helm.values` sprawia, że kontroler — jako cluster-admin — przepisuje ClusterRole samego agenta; wariant drugi podmienia `repoURL` i `destination.namespace` na `kube-system`. **Why:** RBAC nie umie zawęzić `patch` do adnotacji, więc przyznanie go oddaje cały `spec`. Uprawnienie było jawnie zapisane w planie — znalezisko przeważyło nad planem. Refresh okazał się zresztą zbędny: wszystkie Application mają automated sync, a ApplicationSet i tak odpytuje repo (~3 min).

- **2026-07-28 — Wyciek tokenu zamknięty po stronie serwera flagą `--toolsets=core`.** Narzędzie `configuration_view` zwracało pełny in-cluster kubeconfig z żywym JWT ServiceAccountu — użyteczny wprost przeciw API serverowi, czyli **omijający NetworkPolicy**. Pierwotny plan chciał wyłączyć narzędzie po stronie klienta n8n. **Why odrzucone:** to filtr listy narzędzi pokazywanej modelowi, a nie granica — serwer nadal podawałby je na drucie, a NetworkPolicy wpuszcza *cały* namespace `n8n`, więc każdy inny workflow mógłby po nie sięgnąć. `--toolsets=core` usuwa równo to jedno narzędzie (20 → 19) i nic poza nim. Token jest pod-bound, więc w razie wycieku skasowanie poda MCP unieważnia go natychmiast.

- **2026-07-28 — Wyjątek od zasady „granicą jest RBAC, nie flagi MCP".** Token ServiceAccountu to **plik na dysku poda, nie obiekt API** — RBAC nie ma języka, żeby powiedzieć „nie ujawniaj własnego poświadczenia". To jedyne miejsce, gdzie flaga serwera jest jedyną dostępną granicą, i dlatego nie jest cofnięciem zasady.

- **2026-07-28 — NetworkPolicy: ingress do `mcp` wyłącznie z namespace `n8n`.** **Why:** `mcp-k8s` nie ma żadnego uwierzytelniania, więc bez polityki jego tożsamość ServiceAccountu była dostępna dla dowolnego poda w klastrze. Bez tego wszystkie powyższe ograniczenia byłyby granicą wyłącznie dla agenta, a nie dla czegokolwiek innego, co w klastrze działa. (`mcp-actual` szedł inną drogą — bramkował żądania bearer tokenem, dlatego polityki nie potrzebował; 2026-08-30 usunięty z klastra razem z całym Actual.)

## Open questions

- **AppProject `default` nadal ma `sourceRepos`, `destinations` i `clusterResourceWhitelist` na `*`**, przy kontrolerze z `*/*/*`. Cofnięcie `patch` zamknęło ścieżkę *tego* agenta; sama właściwość klastra została nietknięta i czeka. To najpoważniejsza rzecz na tej liście.
- Czy włączyć `selfHeal` na Applications? Zamieniłoby wąskie akcje operacyjne w akcje z natury tymczasowe (dryf cofany w minuty). Cena: własne ręczne `kubectl edit` też byłoby cofane. Świadomie poza zakresem tego projektu.
- NetworkPolicy wpuszcza **cały** namespace `n8n` — łącznie z `n8n-postgres`, `n8n-valkey`, workerami i webhookiem. Zawężenie do samych podów aplikacji wymaga dołożenia `podSelector` do chartu.
- Obraz MCP stoi na tagu `latest` z `IfNotPresent`, na workloadzie, który **jest** granicą bezpieczeństwa. Rekomendowane przypięcie po digeście; niezrobione.
- Brak wykonywalnego testu granicy *sieciowej*, analogicznego do `rbac-check.sh` dla RBAC.
- Token MCP do n8n wygasa i wywraca automatyzację w połowie pracy. Warte rozwiązania systemowego, bo dotknie każdego przyszłego projektu ruszającego n8n programistycznie.

## Links

- [[10_Projects/agent-workforce/_state|Agent Workforce]] — dzieli repo, mastera i filozofię kolejki zatwierdzeń przez PR-y
- [[NOW]] — bieżący kontekst
- [[tools-stack]] — n8n, Kubernetes, Claude Code
- Repo GitOps: `github.com/elanonix/argocd`
- `Personal Assistant (Master)` — n8n `eiuCVFO2GySjtEUB`
- `Budget Agent (Sub-workflow)` — n8n `gFkVVmWKW4Goik05`, wzorzec kształtu sub-agenta (zarchiwizowany 2026-08-30 — Actual Budget usunięty z klastra i z mastera)
