# Linux + NVIDIA RTX 5090 (32GB) — AI Platform Setup Guide v4.2
### Zero-to-Finished · WSL2 + Bare Metal · Self-Healing · Deterministic Routing · Secure by Default

This is the **definitive full guide**. v4.2 fixes the v4.0 regressions by restoring the missing router build steps, restoring full supervisor self-heal logic, and restoring the full `ai-stack` CLI (including WSL/no-systemd fallbacks).

---

## Changelog (v4.2)
- Patched router regex escaping (JSON `\b` boundaries)
- Added router PID early-exit to prevent duplicate uvicorn instances
- Improved `ai-stack` process management: stops systemd supervisor first on `down`, stops Ollama via systemd when available, PID-safe nohup fallback
- Restored **Phase 5.1–5.3**: router venv, dependencies, `router_rules.json`, and a working `router.py`.
- Restored **Phase 6** Docker self-heal loop: checks containers and restarts `docker compose up -d` when needed.
- Restored full **Phase 8** `ai-stack` CLI from v3.9-style behavior: `up|down|doctor|logs` + systemd vs `nohup` fallback for WSL.

---

# PHASE 1 — Dependencies + directory layout

> **Bare metal Ubuntu**: install Docker Engine via apt (below).  
> **WSL2**: install Docker Desktop on Windows and enable WSL integration; in Ubuntu, `docker` should work without installing `docker.io`.

## 1.1 Install baseline packages (both)
```bash
sudo apt update
sudo apt install -y python3 python3-venv curl jq caddy
```

### Bare metal only: Docker Engine
```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
docker run --rm hello-world
```

## 1.2 Create directory structure
```bash
mkdir -p \
  ~/ai-stack/{caddy,searxng,bin} \
  ~/AI_Output/{images,audio,voice,docs} \
  ~/ai-supervisor \
  ~/model-router \
  ~/Backups
```

## 1.3 Permissions (strict + traversable)
```bash
chmod 711 ~/AI_Output
chmod -R 700 ~/AI_Output/{images,audio,voice,docs}
```

---

# PHASE 2 — Ollama install + models

## 2.1 Install Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Bare metal (systemd)
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
```

### WSL fallback (if systemd unavailable)
```bash
nohup ollama serve > ~/ollama.log 2>&1 &
echo $! > /tmp/ollama.pid
```

Validate:
```bash
curl -s http://localhost:11434/api/tags | jq .
```

## 2.2 Pull baseline models
```bash
ollama pull qwen2.5:7b
ollama pull gemma3:27b
ollama pull deepseek-r1:32b
ollama pull nomic-embed-text
```

---

# PHASE 3 — Docker stack (Open WebUI + Chroma + SearxNG)

## 3.1 SearxNG settings (JSON enabled)
```bash
cat > ~/ai-stack/searxng/settings.yml << 'EOF'
use_default_settings: true

search:
  safe_search: 0
  formats:
    - html
    - json

server:
  secret_key: "CHANGE_ME_SEARXNG_SECRET"
  limiter: true
EOF
```

## 3.2 docker-compose.yml
```bash
cat > ~/ai-stack/docker-compose.yml << 'EOF'
services:
  chromadb:
    image: chromadb/chroma:0.5.4
    container_name: chromadb
    read_only: true
    ports:
      - "127.0.0.1:8500:8000"
    volumes:
      - chromadb_data:/chroma/chroma
    environment:
      - ANONYMIZED_TELEMETRY=false
      - ALLOW_RESET=true
    healthcheck:
      test: ["CMD","curl","-f","http://localhost:8000/api/v1/heartbeat"]
      interval: 15s
      timeout: 3s
      retries: 10
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true

  searxng:
    image: searxng/searxng:2026.01.15
    container_name: searxng
    read_only: true
    ports:
      - "127.0.0.1:8888:8080"
    volumes:
      - ./searxng:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://searxng:8080
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true

  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.5.12
    container_name: open-webui
    ports:
      - "127.0.0.1:3000:8080"
    volumes:
      - open_webui_data:/app/backend/data
      - ~/AI_Output/voice:/app/backend/data/voice_samples
    depends_on:
      chromadb:
        condition: service_healthy
      searxng:
        condition: service_started
    environment:
      # Point Open WebUI to the router (Ollama-compatible API)
      - OLLAMA_BASE_URL=http://host.docker.internal:8000

      - VECTOR_DB=chroma
      - CHROMA_HTTP_HOST=chromadb
      - CHROMA_HTTP_PORT=8000

      - WEBUI_AUTH=true

      - ENABLE_RAG_WEB_SEARCH=true
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>&format=json

      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=nomic-embed-text
      - ENABLE_RAG_HYBRID_SEARCH=true
      - CHUNK_SIZE=1000
      - CHUNK_OVERLAP=200
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true

volumes:
  chromadb_data:
  open_webui_data:
EOF
```

Start stack:
```bash
cd ~/ai-stack
docker compose up -d
```

---

# PHASE 4.0 — Router networking (WSL2-aware + persistent)

```bash
mkdir -p ~/model-router

# Detect WSL2
if grep -qi microsoft /proc/version 2>/dev/null || [ -n "${WSL_INTEROP:-}" ]; then
  ENV_KIND="wsl2"
else
  ENV_KIND="baremetal"
fi

# Decide bind + host-reachable address
if [ "$ENV_KIND" = "wsl2" ]; then
  ROUTER_BIND_IP="0.0.0.0"
  ROUTER_HOST_IP="127.0.0.1"
else
  ROUTER_BIND_IP="172.17.0.1"
  ROUTER_HOST_IP="172.17.0.1"

  # Auto-detect docker0 IP if present (custom bridge safe)
  if ip -4 addr show docker0 >/dev/null 2>&1; then
    ROUTER_BIND_IP="$(ip -4 addr show docker0 | awk '/inet /{print $2}' | cut -d/ -f1)"
    ROUTER_HOST_IP="$ROUTER_BIND_IP"
  fi
fi

cat > ~/model-router/.env << EOF
ENV_KIND=$ENV_KIND
ROUTER_BIND_IP=$ROUTER_BIND_IP
ROUTER_HOST_IP=$ROUTER_HOST_IP
ROUTER_PORT=8000
ROUTER_HEALTH_URL=http://$ROUTER_HOST_IP:8000/health
# Optional: set a token to require X-Router-Token on router endpoints
# ROUTER_TOKEN=CHANGEME
EOF

echo "Wrote ~/model-router/.env:"
cat ~/model-router/.env
```

---

# PHASE 4 — Caddy (service-correct path + systemd-safe env)

## 4.1 Write Caddyfile to the service path
```bash
sudo tee /etc/caddy/Caddyfile >/dev/null << 'EOF'
{
  auto_https off
}

:8080 {
  bind 127.0.0.1

  handle_path /router/* {
    reverse_proxy {$ROUTER_HOST_IP}:{$ROUTER_PORT}
  }

  handle /metrics {
    reverse_proxy {$ROUTER_HOST_IP}:{$ROUTER_PORT}
  }

  handle {
    reverse_proxy 127.0.0.1:3000
  }
}
EOF
```

## 4.2 Provide router env to Caddy
```bash
sudo cp ~/model-router/.env /etc/caddy/router.env
sudo chmod 644 /etc/caddy/router.env

sudo mkdir -p /etc/systemd/system/caddy.service.d
sudo tee /etc/systemd/system/caddy.service.d/router-env.conf >/dev/null << 'EOF'
[Service]
EnvironmentFile=/etc/caddy/router.env
EOF

sudo systemctl daemon-reload
sudo systemctl restart caddy
```

### WSL / manual Caddy mode (preserve env for reload & start)
```bash
set -a
source ~/model-router/.env
set +a

sudo -E caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile 2>/dev/null || \
sudo -E caddy start  --config /etc/caddy/Caddyfile --adapter caddyfile
```

---

# PHASE 5 — Router (restore the missing “brain”)

The router exposes an **Ollama-compatible API** to Open WebUI, while applying deterministic routing rules before forwarding requests to the local Ollama daemon at `http://127.0.0.1:11434`.

## 5.1 Create venv + install dependencies
```bash
python3 -m venv ~/model-router/.venv
source ~/model-router/.venv/bin/activate
pip install --upgrade pip
pip install fastapi uvicorn requests pydantic
```

## 5.2 Create router_rules.json (JSON-safe regex escaping)
```bash
cat > ~/model-router/router_rules.json << 'EOF'
{
  "default_model": "qwen2.5:7b",
  "workspaces": {
    "auto": {"lock": false},
    "auto-splunk": {"lock": true, "model": "deepseek-r1:32b"},
    "auto-bigfix": {"lock": true, "model": "gemma3:27b"}
  },
  "regex_routes": [
    {"pattern": "(?i)\\b(splunk|es notable|tstats|cim)\\b", "model": "deepseek-r1:32b", "priority": 10},
    {"pattern": "(?i)\\b(bigfix|fixlet|bes client|relevance)\\b", "model": "gemma3:27b", "priority": 10}
  ],
  "auth_token_env": "ROUTER_TOKEN"
}
EOF
```


## 5.3 Create router.py (Ollama-compatible proxy + deterministic routing)
```bash
cat > ~/model-router/router.py << 'EOF'
import json
import os
import re
from typing import Any, Dict, Optional, Tuple

import requests
from fastapi import FastAPI, Header, HTTPException, Request
from fastapi.responses import JSONResponse, StreamingResponse

OLLAMA_UPSTREAM = os.environ.get("OLLAMA_UPSTREAM", "http://127.0.0.1:11434")
RULES_PATH = os.environ.get("ROUTER_RULES", os.path.expanduser("~/model-router/router_rules.json"))
ROUTER_TOKEN_ENV = "ROUTER_TOKEN"

app = FastAPI(title="Local Model Router", version="1.0")

def load_rules() -> Dict[str, Any]:
    with open(RULES_PATH, "r", encoding="utf-8") as f:
        return json.load(f)

def require_token(x_router_token: Optional[str]) -> None:
    rules = load_rules()
    token_env = rules.get("auth_token_env", ROUTER_TOKEN_ENV)
    expected = os.environ.get(token_env)
    if expected:
        if not x_router_token or x_router_token.strip() != expected.strip():
            raise HTTPException(status_code=401, detail="Unauthorized")

def extract_text(payload: Dict[str, Any]) -> str:
    # Ollama chat: {"messages":[{"role":"user","content":"..."}], ...}
    if isinstance(payload.get("messages"), list):
        parts = []
        for m in payload["messages"]:
            c = m.get("content")
            if isinstance(c, str):
                parts.append(c)
        return "\n".join(parts)
    # Ollama generate: {"prompt":"..."}
    if isinstance(payload.get("prompt"), str):
        return payload["prompt"]
    return ""

def manual_override(text: str) -> Optional[str]:
    # @model:deepseek-r1:32b (first match wins)
    m = re.search(r"@model\s*:\s*([A-Za-z0-9_.:-]+)", text)
    return m.group(1) if m else None

def pick_model(payload: Dict[str, Any]) -> str:
    rules = load_rules()
    default_model = rules.get("default_model", "qwen2.5:7b")

    # If caller explicitly set model in JSON, honor it (allows UI-side selection)
    if isinstance(payload.get("model"), str) and payload["model"].strip():
        return payload["model"].strip()

    text = extract_text(payload)

    # 1) @model override
    mo = manual_override(text)
    if mo:
        return mo

    # 2) workspace lock (optional hint via header or payload)
    #   Open WebUI doesn’t consistently send this; keep it as a reserved extension.
    workspace = payload.get("workspace") or payload.get("metadata", {}).get("workspace")
    if isinstance(workspace, str):
        ws = rules.get("workspaces", {}).get(workspace)
        if ws and ws.get("lock") and ws.get("model"):
            return ws["model"]

    # 3) regex routes by priority
    routes = rules.get("regex_routes", [])
    # sort high->low
    routes = sorted(routes, key=lambda r: int(r.get("priority", 0)), reverse=True)
    for r in routes:
        pat = r.get("pattern")
        mdl = r.get("model")
        if not (isinstance(pat, str) and isinstance(mdl, str)):
            continue
        try:
            if re.search(pat, text or ""):
                return mdl
        except re.error:
            # ignore bad pattern rather than crashing
            continue

    # 4) default
    return default_model

def stream_upstream(method: str, path: str, headers: Dict[str, str], body: bytes):
    url = f"{OLLAMA_UPSTREAM}{path}"
    with requests.request(method, url, headers=headers, data=body, stream=True, timeout=600) as r:
        r.raise_for_status()
        def gen():
            for chunk in r.iter_content(chunk_size=8192):
                if chunk:
                    yield chunk
        return StreamingResponse(gen(), status_code=r.status_code, media_type=r.headers.get("content-type"))

@app.get("/health")
def health():
    # lightweight check; don’t block on upstream
    return {"ok": True}

@app.api_route("/{full_path:path}", methods=["GET","POST","PUT","PATCH","DELETE"])
async def proxy(full_path: str, request: Request, x_router_token: Optional[str] = Header(default=None)):
    # Optional auth
    require_token(x_router_token)

    path = "/" + full_path

    # Pass through headers but strip hop-by-hop items
    fwd_headers = {}
    for k, v in request.headers.items():
        lk = k.lower()
        if lk in ("host", "content-length", "connection"):
            continue
        fwd_headers[k] = v

    body = await request.body()

    # Intercept model selection for Ollama endpoints that accept a model
    if path in ("/api/chat", "/api/generate"):
        try:
            payload = json.loads(body.decode("utf-8") if body else "{}")
        except json.JSONDecodeError:
            payload = {}
        payload["model"] = pick_model(payload)
        body = json.dumps(payload).encode("utf-8")
        fwd_headers["Content-Type"] = "application/json"

    # Stream for chat/generate to preserve Open WebUI streaming
    if request.method.upper() == "POST" and path in ("/api/chat", "/api/generate"):
        return stream_upstream(request.method, path, fwd_headers, body)

    # Non-stream requests
    url = f"{OLLAMA_UPSTREAM}{path}"
    r = requests.request(request.method, url, headers=fwd_headers, data=body, timeout=600)
    # Response
    resp_headers = {}
    for k, v in r.headers.items():
        lk = k.lower()
        if lk in ("content-encoding", "transfer-encoding", "connection"):
            continue
        resp_headers[k] = v
    return JSONResponse(status_code=r.status_code, content=r.json() if "application/json" in r.headers.get("content-type","") else {"raw": r.text}, headers=resp_headers)
EOF
```

## 5.4 Create/overwrite launch_router.sh (secure fallback)
```bash
cat > ~/launch_router.sh << 'EOF'
#!/bin/bash
set -e

# Early exit if router is already running
if [ -f /tmp/router.pid ] && kill -0 "$(cat /tmp/router.pid)" 2>/dev/null; then
  echo "[router] already running (PID $(cat /tmp/router.pid))."
  exit 0
fi

# Persistent config (works under systemd too)
if [ -f ~/model-router/.env ]; then
  set -a
  source ~/model-router/.env
  set +a
fi

source ~/model-router/.venv/bin/activate

until curl -s http://localhost:11434/api/tags >/dev/null; do
  echo "[router] waiting for Ollama..."
  sleep 2
done

# FAIL-CLOSED DEFAULT:
# If .env is missing, default to docker bridge IP to avoid LAN exposure.
ROUTER_BIND_IP="${ROUTER_BIND_IP:-172.17.0.1}"
ROUTER_PORT="${ROUTER_PORT:-8000}"

nohup uvicorn router:app --host "$ROUTER_BIND_IP" --port "$ROUTER_PORT" --app-dir ~/model-router \
  > ~/model-router/router.log 2>&1 &
echo $! > /tmp/router.pid
echo "[router] started (PID $(cat /tmp/router.pid)) on $ROUTER_BIND_IP:$ROUTER_PORT"
EOF

chmod +x ~/launch_router.sh
```


Validate router:
```bash
set -a; source ~/model-router/.env; set +a
~/launch_router.sh
curl -s "$ROUTER_HEALTH_URL" | jq .
```

---

# PHASE 6 — Supervisor (sudoers + full self-heal loop)

## 6.1 Allow supervisor to restart Ollama without password
```bash
sudo tee /etc/sudoers.d/ai-supervisor-ollama >/dev/null << 'EOF'
Defaults:%sudo !requiretty
%sudo ALL=(root) NOPASSWD: /bin/systemctl restart ollama, /bin/systemctl status ollama
EOF

sudo chmod 440 /etc/sudoers.d/ai-supervisor-ollama
```

## 6.2 Create/overwrite supervisor.sh (FULL logic restored)
```bash
mkdir -p ~/ai-supervisor

cat > ~/ai-supervisor/supervisor.sh << 'EOF'
#!/bin/bash
set -u

LOG=~/ai-supervisor/supervisor.log
FAILS=0

log(){ echo "[$(date -Is)] $*" | tee -a "$LOG" >/dev/null; }

# Load persistent router settings (systemd-safe)
if [ -f ~/model-router/.env ]; then
  set -a
  source ~/model-router/.env
  set +a
fi

ROUTER_HEALTH_URL="${ROUTER_HEALTH_URL:-http://127.0.0.1:8000/health}"

start_router(){
  if [ -f /tmp/router.pid ] && kill -0 "$(cat /tmp/router.pid)" 2>/dev/null; then
    return
  fi
  log "Starting router..."
  ~/launch_router.sh || true
}

while true; do
  # 1) Ollama health
  if ! curl -s http://localhost:11434/api/tags >/dev/null; then
    log "Ollama unhealthy."
    if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files | grep -q '^ollama\.service'; then
      log "Restarting ollama.service via systemctl."
      sudo /bin/systemctl restart ollama || true
    else
      log "Restarting Ollama via nohup."
      pkill -f "ollama serve" 2>/dev/null || true
      nohup ollama serve > ~/ollama.log 2>&1 & echo $! > /tmp/ollama.pid
    fi
    FAILS=$((FAILS+1))
    sleep $((FAILS*5))
    continue
  fi

  # 2) Router health
  if ! curl -s "$ROUTER_HEALTH_URL" >/dev/null; then
    log "Router unhealthy at $ROUTER_HEALTH_URL. Restarting."
    [ -f /tmp/router.pid ] && kill "$(cat /tmp/router.pid)" 2>/dev/null || true
    start_router
    FAILS=$((FAILS+1))
    sleep $((FAILS*5))
    continue
  fi

  # 3) Docker stack presence (RESTORED)
  if ! docker ps --format '{{.Names}}' | grep -q '^open-webui$'; then
    log "Docker stack missing or Open WebUI down. Running docker compose up -d."
    (cd ~/ai-stack && docker compose up -d) || true
    FAILS=$((FAILS+1))
    sleep $((FAILS*5))
    continue
  fi

  # Optional: ensure core services exist (non-fatal if missing)
  docker ps --format '{{.Names}}' | grep -q '^chromadb$' || log "WARN: chromadb not running"
  docker ps --format '{{.Names}}' | grep -q '^searxng$'  || log "WARN: searxng not running"

  FAILS=0
  sleep 20
done
EOF

chmod +x ~/ai-supervisor/supervisor.sh
```

---

# PHASE 7 — Zero-touch boot (bare metal systemd)

```bash
sudo tee /etc/systemd/system/ai-supervisor.service >/dev/null << EOF
[Unit]
Description=AI Platform Supervisor
After=network.target docker.service ollama.service caddy.service
Requires=docker.service ollama.service caddy.service

[Service]
User=$USER
ExecStart=/home/$USER/ai-supervisor/supervisor.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ai-supervisor
sudo systemctl start ai-supervisor
```

---

# PHASE 8 — Unified CLI (`ai-stack`) (FULL script restored; WSL-safe)

```bash
cat > ~/ai-stack/bin/ai-stack << 'EOF'
#!/bin/bash
set -e

# Load persistent router settings (systemd-safe)
if [ -f ~/model-router/.env ]; then
  set -a
  source ~/model-router/.env
  set +a
fi
ROUTER_HEALTH_URL="${ROUTER_HEALTH_URL:-http://127.0.0.1:8000/health}"

CMD="${1:-help}"

case "$CMD" in
  up)
    echo "[ai-stack] Starting platform..."
    (cd ~/ai-stack && docker compose up -d)
    ~/launch_router.sh || true

    # Start supervisor via systemd if available; otherwise use safe nohup fallback (WSL)
    if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files 2>/dev/null | grep -q '^ai-supervisor\.service'; then
      sudo systemctl start ai-supervisor || true
    else
      if [ ! -f /tmp/ai-supervisor.pid ] || ! kill -0 "$(cat /tmp/ai-supervisor.pid)" 2>/dev/null; then
        nohup ~/ai-supervisor/supervisor.sh > ~/ai-supervisor/supervisor.log 2>&1 & echo $! > /tmp/ai-supervisor.pid
      else
        echo "[ai-stack] Supervisor already running."
      fi
    fi
    ;;
  down)
    echo "[ai-stack] Stopping platform..."

    # Stop the systemd supervisor first so it doesn't resurrect services
    if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files 2>/dev/null | grep -q '^ai-supervisor\.service'; then
      sudo systemctl stop ai-supervisor || true
    fi

    (cd ~/ai-stack && docker compose down) || true
    [ -f /tmp/router.pid ] && kill "$(cat /tmp/router.pid)" 2>/dev/null || true
    [ -f /tmp/ai-supervisor.pid ] && kill "$(cat /tmp/ai-supervisor.pid)" 2>/dev/null || true

    # Stop Ollama cleanly via systemd if possible, else pkill
    if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files 2>/dev/null | grep -q '^ollama\.service'; then
      sudo systemctl stop ollama || true
    else
      pkill -f "ollama serve" 2>/dev/null || true
    fi
    ;;
  doctor)
    echo "=== AI STACK DOCTOR ==="
    echo "-- GPU --"
    nvidia-smi | head -n 3 2>/dev/null || echo "nvidia-smi not available (WSL/Mac path or driver not installed)"
    echo "-- Ollama --"
    curl -s http://localhost:11434/api/tags >/dev/null && echo "Ollama OK" || echo "Ollama FAIL"
    echo "-- Router --"
    curl -s "$ROUTER_HEALTH_URL" >/dev/null && echo "Router OK" || echo "Router FAIL"
    echo "-- Docker --"
    docker ps --format '{{.Names}}	{{.Status}}' | grep -E 'open-webui|chromadb|searxng' || true
    echo "-- UI --"
    curl -s http://localhost:8080 >/dev/null && echo "UI OK (http://localhost:8080)" || echo "UI FAIL"
    ;;
  logs)
    tail -n 200 ~/ai-supervisor/supervisor.log 2>/dev/null || true
    tail -n 200 ~/model-router/router.log 2>/dev/null || true
    ;;
  *)
    echo "Usage: ai-stack {up|down|doctor|logs}"
    ;;
esac
EOF

chmod +x ~/ai-stack/bin/ai-stack
sudo ln -sf ~/ai-stack/bin/ai-stack /usr/local/bin/ai-stack
```

---

# PHASE 9 — Hardware architecture alignment note (Mac Studio / Apple Silicon)

This guide is tuned for **Linux + NVIDIA + CUDA**.

For **Mac Studio (Apple Silicon)**, the foundation changes:
- No `nvidia-smi`
- Ollama uses **Metal / MPS**
- Docker Desktop runs inside a VM; `docker0` behaves differently or is not present
- Router binding and host reachability must be aligned to Docker Desktop networking

Recommendation: create a separate “Mac Studio Appliance” variant that:
- removes WSL2 instructions
- uses macOS-native Docker Desktop installation flow
- uses `ollama` for Apple Silicon + MPS
- changes router bind defaults to safe localhost patterns compatible with Docker Desktop

---

# Final validation
```bash
ai-stack up
ai-stack doctor
curl -s http://localhost:8080 | head
```

---

## End of guide — v4.2
