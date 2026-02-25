# Your RTX 5090 Has 32GB of VRAM. Stop Wasting It.

> *The M4 Mini Pro guide started it. This is the Linux answer — same idea, different physics. CUDA instead of MPS. 32GB of dedicated GPU memory instead of unified RAM. Bare metal Ubuntu or WSL2. A self-healing supervisor so the stack comes back on its own when things go sideways. And the same single URL at the end.*

From a fresh Linux install to a running AI stack, fully automated, no cloud required.

---

## What You're Building

One URL. Everything local. Zero subscriptions. Zero API keys. Zero data leaving your machine.

Navigate to `http://localhost:8080` and you get Open WebUI backed by:

- **A model router that picks the right LLM automatically** — Splunk questions go to `deepseek-r1:32b`, BigFix questions go to `gemma3:27b`, everything else routes to `qwen2.5:7b` unless you override it
- **RAG over your own documents** — upload your policy PDFs, query them by name, get answers sourced from your own content
- **Live web search** from SearxNG running locally — no Bing API key, no Google dependency
- **ChromaDB for vector storage** — your document embeddings persist on disk across restarts
- **A self-healing supervisor** that monitors Ollama, the router, and the Docker stack — restarts whatever falls over, logs what it did, backs off on repeated failures

This isn't just a compose file. The router has routing rules. The supervisor has real health checks. The `ai-stack` CLI gives you a single command to bring everything up, down, check health, and tail logs. It works the same whether you're on bare metal Ubuntu or WSL2.

**This is a 9-phase step-by-step guide.** Every phase has complete commands and configuration. You will have a running system.

---

## By the Numbers

| | |
|---|---|
| **9** phases — fresh Linux to full running stack |
| **4** baseline models pulled at setup — add more from Ollama at any time |
| **4** `ai-stack` commands — `up`, `down`, `doctor`, `logs` |
| **2** deployment targets — bare metal Ubuntu and WSL2, same guide |
| **32GB** dedicated VRAM — the RTX 5090 runs large models without touching system RAM |
| **$0/month** in API costs after setup |

---

## What You Need

| Component | Requirement |
|-----------|-------------|
| GPU | NVIDIA RTX 5090 |
| VRAM | 32GB dedicated |
| OS | Ubuntu (bare metal) or WSL2 on Windows |
| Runtime | Docker Engine (bare metal) or Docker Desktop with WSL integration |
| Packages | `python3`, `python3-venv`, `curl`, `jq`, `caddy` |

> **Bare metal vs. WSL2:** The guide handles both. The router networking setup auto-detects which environment you're in and configures bind addresses accordingly — `172.17.0.1` on bare metal (docker0 bridge), `0.0.0.0` on WSL2. Caddy reads these from an env file so you never have to think about it again.

---

## Start Here

| Document | What It Covers |
|----------|----------------|
| [**Linux + NVIDIA 5090 AI Platform Setup Guide v4.2**](docs/Linux_NVIDIA_5090_AI_Platform_Setup_Guide_v4_2.md) | All 9 phases from fresh install to running stack — complete commands, configs, and WSL2/bare metal variants |

---

## How It's Laid Out

```
                        http://localhost:8080
                                 │
                        ┌────────▼────────┐
                        │      Caddy      │  One URL to rule them all
                        │     :8080       │  127.0.0.1 bind · env-driven routing
                        └────────┬────────┘
               ┌─────────────────┼─────────────────┐
               │                 │                 │
       ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐
       │   Open WebUI  │ │    Router     │ │   SearxNG     │
       │    :3000      │ │    :8000      │ │    :8888      │
       │  Chat + RAG   │ │  Model Brain  │ │  Local Search │
       └───────┬───────┘ └───────┬───────┘ └───────────────┘
               │                 │
       ┌───────▼───────┐ ┌───────▼─────────────────────────┐
       │   ChromaDB    │ │   Ollama  :11434                 │
       │    :8500      │ │   qwen2.5:7b  (default)          │
       │  Your Docs    │ │   gemma3:27b  (BigFix routing)   │
       └───────────────┘ │   deepseek-r1:32b (Splunk/reason)│
                         │   nomic-embed-text (RAG embeds)  │
                         └──────────────────────────────────┘

  Supervisor — always watching, always recovering
    ai-supervisor.sh   Polls Ollama · Router · Docker stack every 20s
                       Restarts what's down · Backs off on repeat failures
                       Logs every action to ~/ai-supervisor/supervisor.log

  CLI — one command for everything
    ai-stack up        Starts Docker stack, router, supervisor
    ai-stack down      Stops everything cleanly, systemd-aware
    ai-stack doctor    Health check: GPU, Ollama, router, containers, UI
    ai-stack logs      Tails supervisor and router logs
```

---

## The 9 Phases

| Phase | What Happens |
|-------|-------------|
| 1 | Dependencies + directory layout — apt packages, Docker Engine (bare metal) or Docker Desktop (WSL2), output directories with strict permissions |
| 2 | Ollama + baseline models — install, enable as systemd service or WSL nohup fallback, pull four models |
| 3 | Docker stack — `docker-compose.yml` for Open WebUI, ChromaDB, SearxNG with security hardening (`no-new-privileges`, `read_only`) |
| 4.0 | Router networking — WSL2 auto-detection, `.env` file with bind/host addresses, persistent across restarts |
| 4 | Caddy — writes to `/etc/caddy/Caddyfile`, reads router env via systemd drop-in, WSL manual mode included |
| 5 | Model Router — Python venv, `router_rules.json` with keyword routing, full `router.py` Ollama-compatible proxy |
| 6 | Supervisor — sudoers for passwordless Ollama restart, full self-heal loop: Ollama + router + Docker stack |
| 7 | Zero-touch boot — systemd unit for the supervisor, enabled at install, starts after `docker.service` and `ollama.service` |
| 8 | `ai-stack` CLI — unified up/down/doctor/logs with systemd awareness and WSL nohup fallback |

---

## The Model Router — Deterministic Before It Hits Ollama

The router is a FastAPI proxy that sits between Open WebUI and Ollama. It reads `router_rules.json`, scores your input against regex patterns, and rewrites the model field in the request before forwarding:

| You type... | Router sends... | Rule |
|-----------|----------------|------|
| Splunk, ES notable, tstats, CIM | `deepseek-r1:32b` | Regex keyword hit, priority 10 |
| BigFix, fixlet, BES client, relevance | `gemma3:27b` | Regex keyword hit, priority 10 |
| `@model:gemma3:27b` anywhere | `gemma3:27b` | Manual override, always wins |
| Anything else | `qwen2.5:7b` | Default fallback |

Workspace locking is also supported — set a workspace in your payload and the router honors it deterministically. Add new routing rules to `router_rules.json` without touching the router code.

---

## The Supervisor — It Fixes Itself

The self-healing loop runs every 20 seconds and checks three things in order:

1. **Ollama** — if `/api/tags` doesn't respond, restart via `systemctl restart ollama` (bare metal) or `pkill`/`nohup` (WSL)
2. **Router** — if `/health` doesn't respond, kill the PID and relaunch
3. **Docker stack** — if `open-webui` isn't in `docker ps`, run `docker compose up -d`

If a service keeps failing, the supervisor backs off with escalating sleep delays so it doesn't spin your disk. Every action is timestamped and appended to `~/ai-supervisor/supervisor.log`.

On bare metal, the supervisor runs as a systemd service and starts automatically at boot. On WSL2, the `ai-stack up` command launches it via `nohup`.

---

## Security Defaults

The stack is built tighter than a typical Docker tutorial:

- All containers run with `no-new-privileges: true`
- ChromaDB and SearxNG containers use `read_only: true` filesystems
- Caddy binds to `127.0.0.1` only — not accessible from the network without explicit change
- The router defaults to the docker bridge IP (`172.17.0.1`) if `.env` is missing, preventing accidental LAN exposure
- Strict directory permissions on `~/AI_Output/` — `711` on parent, `700` on subdirectories
- Optional `ROUTER_TOKEN` header auth — set it in `.env` and the router requires it on all requests

---

## WSL2 Notes

The guide handles WSL2 throughout — it's not an afterthought. Key differences are documented at each phase:

- Docker: use Docker Desktop with WSL integration instead of installing `docker.io`
- Ollama: use `nohup ollama serve &` instead of systemd if systemd isn't available
- Router: binds to `0.0.0.0` in WSL2 so Docker containers can reach it via `host.docker.internal`
- Caddy: manual env-sourced start command instead of `systemctl restart`
- Supervisor: `nohup` launch via `ai-stack up` instead of systemd service

---

## Everything Stays on Your Machine

No subscriptions. No API keys for AI services. No telemetry to vendors. No model inference leaving the box. Your documents, your queries, your infrastructure notes — none of it goes anywhere.

The entire stack runs locally. The supervisor makes sure it stays running. The CLI makes sure you can manage it without memorizing Docker commands.

---

## Versioning

| Document | Version | Update |
|----------|---------|--------|
| Linux + NVIDIA 5090 AI Platform Setup Guide | **v4.2** | Feb 2026 — router, supervisor, and CLI fully restored after v4.0 regressions |

Every change is logged in the changelog at the top of the [Setup Guide](docs/Linux_NVIDIA_5090_AI_Platform_Setup_Guide_v4_2.md).

---

## License

MIT — see [LICENSE](LICENSE). Build on it, adapt it, share it.
