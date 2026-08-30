---
type: inbox
domain: projects
status: inbox
created: 2026-07-28
updated: 2026-07-28
tags: [n8n, llm, cliproxyapi, homelab, agenci]
aliases: [CLIProxyAPI w n8n, mapping modeli]
---

# Modele LLM w n8n — jak tego używamy

Od 2026-07-28 agenci w n8n nie używają płatnego API per token. Wszystkie wywołania LLM idą przez **CLIProxyAPI** — self-hosted proxy w klastrze (namespace `cliproxyapi`), które wystawia subskrypcje Claude Pro/Max, ChatGPT Team i Gemini (Antigravity) jako API zgodne z OpenAI.

## Jak to działa

- W n8n jest jeden credential typu OpenAI o nazwie **CLIProxyAPI**: Base URL `http://cliproxyapi.cliproxyapi.svc.cluster.local:8317/v1` + klucz API z `gitops/apps/cliproxyapi/secret-config.yaml`.
- Zwykłe nody **OpenAI Chat Model** korzystają z tego credentiala; dropdown modeli pokazuje modele wszystkich trzech dostawców naraz.
- Routing po nazwie modelu: `claude-*` → subskrypcja Claude, `gpt-*` → ChatGPT (Codex), `gemini-*` → Google (Antigravity). Model wybiera się per node — proxy nic nie decyduje samo.
- Tokeny OAuth leżą na PVC `cliproxyapi-auth` i odświeżają się same; po wygaśnięciu refresh tokena trzeba powtórzyć logowanie lokalnie (instrukcja w README appki w repo argocd).

## Przydział modeli w agentach

| Workflow | Model | Powód |
|---|---|---|
| Personal Assistant (Master) | `gpt-5.5` | orkiestracja + tool-calling, najczęstsze wywołania |
| Obsidian Agent | `gemini-3.6-flash-high` | lekki capture notatek, szybki, rozkłada limity |
| English Coach | `claude-sonnet-5` | jakość języka, naturalna proza (czytana TTS-em) |
| English Daily | — | bez własnego LLM, woła English Coach |

## Czego proxy NIE robi

- **STT zostaje lokalne** (serwis voice w klastrze): subskrypcje nie wystawiają endpointu transkrypcji (`/v1/audio/transcriptions` to produkt API). Głos → własny STT → tekst → modele przez proxy.
- Smoke test: workflow **„CLIProxyAPI - test polaczenia"** w n8n (manual trigger).

## Zastrzeżenia

- Limity subskrypcyjne (okna czasowe) obowiązują — to nie jest nielimitowane API.
- Ścieżka `claude-*` przez proxy formalnie łamie ToS Anthropic (ryzyko bana konta) — dlatego ograniczona do coacha; awaryjnie podmienić na `gpt-5.6-sol`.
- Hardening proxy: management API wyłączone (CVE-2026-8081), panel WebUI wyłączony, NetworkPolicy ingress tylko z ns `n8n`.

Powiązane: [[cliproxyapi]], [[n8n]], [[personal-assistant]]
