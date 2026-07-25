# Automation Stack

n8n workflow automation, local LLM inference, push notifications, and container log monitoring.

## Services

### n8n

Visual workflow engine for automating tasks and integrations. Runs with basic auth enabled.

- **Data directory** — `./n8n/` (workflows, credentials, data)
- **Webhook URL** — configured via `N8N_URL` environment variable (used by Wazuh SOAR integration)

### Ollama

Local LLM inference, Pi-optimized for constrained hardware:

- `MAX_QUEUE=512` — queue size for concurrent requests
- `MAX_VRAM=2GB` — VRAM allocation
- `FLASH_ATTENTION=1` — flash attention optimization
- `MAX_LOADED_MODELS=1` — only one model in memory at a time
- `KEEP_ALIVE=2m` — keep loaded model in memory for 2 minutes after last request

### Open WebUI

Feature-rich web UI for Ollama with RAG, chat, and extensions. Built with a custom Dockerfile (`./ollama/Dockerfile`) optimized for Raspberry Pi.

### NTFY

Self-hosted push notification server with web dashboard and CLI. Used by the automation stack and Wazuh SOAR integration for alert delivery.

- **Cache** — `./ntfy/ntfy/cache/ntfy/`
- **Config** — `./ntfy/ntfy/config/`
- **Database** — `./ntfy/ntfy/db/`

### LoggiFly

Monitors container logs and sends alerts to NTFY. Uses a read-only socket proxy (`11notes/socket-proxy:2`) for safe Docker socket access.

## Setup

1. Edit `.env` with your configuration.

2. Start the stack:
   ```bash
   docker compose up -d
   ```

3. Access services via Tailscale DNS:
   - **n8n** — `n8n.ts.mydomain.com`
   - **Ollama** — `ollama.ts.mydomain.com`
   - **Open WebUI** — `open-webui.ts.mydomain.com`
   - **NTFY** — `ntfy.ts.mydomain.com` (HTTP), `https://ntfy.ts.mydomain.com` (HTTPS)

n8n (workflow engine)
    ↓
Ollama (LLM inference) ←→ Open WebUI (chat UI, RAG)
    ↓
NTFY (push notifications) ←→ LoggiFly (log monitoring)
    ↓
loggifly-socket-proxy (read-only Docker socket)


All services communicate over the `automation-stack` Docker network (`172.31.0.0/16`). The Wazuh SOAR integration triggers n8n workflows via webhook, which then route alerts to Ollama for AI triage or to NTFY for push notifications.

### Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_PORT` | `5678` | n8n web interface port |
| `N8N_USER` | *(required)* | n8n basic auth username |
| `N8N_PW` | *(required)* | n8n basic auth password |
| `N8N_URL` | *(required)* | n8n webhook URL (publicly accessible) |
| `OLLAMA_PORT` | `11434` | Ollama API port |
| `OLLAMA_DOCKER_TAG` | `latest` | Ollama Docker image tag |
| `OPEN_WEBUI_PORT` | `3020` | Open WebUI port |
| `WEBUI_SECRET_KEY` | *(required)* | Open WebUI secret key |
| `NTFY_HTTP_PORT` | `8015` | NTFY HTTP port |
| `NTFY_HTTPS_PORT` | `9015` | NTFY HTTPS port |
| `AUTOMATION_SUBNET` | `172.31.0.0/16` | Docker network subnet |

