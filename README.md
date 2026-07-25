# Homelab

Documentation of my homelab running across a Raspberry Pi 5 and a DigitalOcean VPS (Pangolin stack), connected via Tailscale mesh. Services are managed with Docker Compose, DNS is handled by Cloudflare, and a MikroTik hEX S router enforces network segmentation.

Each service lives in its own directory with a standalone `docker-compose.yml` and a `.env` file. See `.env.example` for all variables with generation instructions.

## Architecture

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TB
    subgraph DNS["Cloudflare DNS"]
        direction LR
        A1["*.example.com<br/>A record"] --> A2["DigitalOcean VPS"]
        B1["*.ts.example.com<br/>A record"] --> B2["Pi Tailscale IP"]
    end
    subgraph VPS["DigitalOcean VPS (Pangolin stack)"]
        direction TB
        Traefik["Traefik + Bouncer<br/>:80, :443"]
        CrowdSec["CrowdSec"]
        Pangolin["Pangolin<br/>:3000-3002"]
        Gerbil["Gerbil<br/>:51820"]
        Traefik -- "Newt tunnel<br/>(WireGuard)" --> Pangolin
        Traefik --- CrowdSec
        Pangolin --- Gerbil
    end
    subgraph PI["Raspberry Pi 5 (16GB, NVMe)"]
        direction TB
        Tailscale["Tailscale (host)"]
        Nginx["Nginx Proxy<br/>:80, :443"]
        subgraph APP["Application Layer"]
            direction TB
            subgraph APPROW1[" "]
                direction LR
                Authentik["Authentik<br/>:9000"]
                Vaultwarden["Vaultwarden<br/>:8000"]
                AdGuard["AdGuard Home<br/>:53, :443"]
            end
            subgraph APPROW2[" "]
                direction LR
                Portainer["Portainer<br/>:9443"]
                Guacamole["Guacamole<br/>:8040"]
                Automation["Automation<br/>n8n / Ollama"]
            end
        end
        Tailscale --> Nginx
        Nginx --> APP
    end
    DNS --> VPS
    VPS -- "Newt tunnel" --> Nginx
    style DNS fill:#1f2937,stroke:#58a6ff,stroke-width:1px,color:#c9d1d9
    style VPS fill:#2d2410,stroke:#d29922,stroke-width:1px,color:#c9d1d9
    style PI fill:#0d2818,stroke:#3fb950,stroke-width:1px,color:#c9d1d9
    style APP fill:#161b22,stroke:#8b949e,stroke-width:1px,color:#c9d1d9
    style APPROW1 fill:none,stroke:none
    style APPROW2 fill:none,stroke:none
```

## Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| **nginx** | jc21/nginx-proxy-manager | host | Reverse proxy + Let's Encrypt SSL |
| **goaccess** | justsky/goaccess-for-nginxproxymanager | 7880 | Web traffic analytics dashboard |
| **authentik** | ghcr.io/goauthentik/server | 9000, 9443 | Identity provider (SSO, MFA, OIDC) |
| **adguardhome** | adguard/adguardhome | 53, 4433 | Network-wide DNS ad blocking |
| **vaultwarden** | vaultwarden/server | 8000 | Password manager (Bitwarden compatible) |
| **tailscale** | tailscale/tailscale | host | Zero-trust VPN node |
| **guacamole** | guacamole/guacamole + postgres:alpine | 8040 | Remote desktop gateway (VNC/RDP/SSH) |
| **loggifly** | loggifly/loggifly | — | Container log monitor → NTFY alerts |
| **n8n** | n8nio/n8n | 5678 | Workflow automation |
| **ollama** | ollama/ollama | 11434 | Local LLM inference |
| **open-webui** | ghcr.io/open-webui/open-webui | 3020 | Web UI for Ollama |
| **pangolin** | fosrl/pangolin + crowdsecurity/crowdsec | 3000, 8080 | Security dashboard + IDS (VPS) |
| **wazuh** | wazuh/wazuh-manager + wazuh-agent | 1514, 1515, 55000 | XDR/SIEM — endpoint monitoring, FIM |
| **nessus** | tenable/nessus:latest-oracle | 8834 | Vulnerability scanner |

### Identity and Access Management

#### Authentik

Open-source Identity Provider with SSO, MFA, and OIDC. Supports OpenID Connect for SSO integration with other services (Guacamole, Vaultwarden).
[→ Details](pi/docker/authentik/)

### Remote Access Services

#### Apache Guacamole

Clientless remote desktop gateway (VNC/RDP/SSH) via HTML5 with Authentik SSO support.
[→ Details](pi/docker/guacamole/)

### Web Traffic Analytics

#### GoAccess

Real-time analytics dashboard for Nginx Proxy Manager access logs.
[→ Details](pi/docker/nginx/)

### Security Stack

#### Wazuh (SIEM + XDR)

XDR/SIEM with FIM, CIS compliance, and n8n-powered SOAR alert triage.
[→ Details](wazuh/)

#### Tenable Nessus

Self-hosted vulnerability scanner detecting flaws, misconfigurations, and insecure protocols.
[→ Details](https://mluchetti.com/docs/security/nessus/)

### Automation Stack

#### n8n + Ollama + Open WebUI

Workflow automation, local LLM inference (Pi-optimized), and chat UI with RAG.
[→ Details](pi/docker/automation/)

#### NTFY + LoggiFly

Push notification server for alert delivery, with container log monitoring.
[→ Details](pi/docker/automation/)

### VPS Stack

#### Pangolin

DigitalOcean VPS stack with CrowdSec IDS, Traefik reverse proxy, and Pangolin security dashboard.
[→ Details](https://mluchetti.com/docs/pangolin/)

## Networking

### DNS (Cloudflare)

Two wildcard DNS records route traffic:
- `*.example.com` → DigitalOcean VPS public IP (public-facing services via Pangolin)
- `*.ts.example.com` → Pi's Tailscale IP (private services over Tailscale mesh)

### Tailscale Mesh

Tailscale runs in host network mode on the Pi, providing zero-trust mesh VPN. The VPS connects via a Newt (Pangolin's WireGuard agent) tunnel.

Tailnet nodes are organized into tags:

| Tag | Host |
|-----|------|
| `tag:clients` | Workstations / mobile devices |
| `tag:pi` | Raspberry Pi 5 (homelab) |
| `tag:servers` | DigitalOcean VPS (Pangolin stack) |

ACL rules are defined in [`tailscale/acl.json`](tailscale/acl.json). Key policies:

- **DNS** — clients → Pi (tcp/udp:53)
- **HTTP/HTTPS** — clients → servers (tcp:80, 443)
- **SSH** — clients → Pi (tcp:22)
- **Wazuh** — clients → Pi (tcp:1514, 1515, 55000)
- **Subnet route** — clients → 10.0.30.0/24 (all ports)
- **Exit node** — clients → internet (all traffic)

### Network Segmentation (MikroTik)

The Pi is isolated in its own VLAN (VLAN 20, DMZ) on the MikroTik hEX S router. Firewall rules enforce LAN↔Pi SSH/DNS only, block WAN→Pi DNS, and block all other cross-VLAN traffic.

## Prerequisites

- Raspberry Pi 5 (16GB, NVMe) or similar Linux host
- DigitalOcean droplet (for Pangolin stack)
- Tailscale account
- Docker
- Domain names managed in Cloudflare
- Cloudflare DNS API token (for Let's Encrypt)

## Setup

1. Clone this repository:
   ```bash
   git clone <repo-url>
   cd homelab
   ```

2. Review `.env.example` for all available variables, then edit the `.env` file in each service directory:
   ```bash
   cd pi/docker/vaultwarden
   nano .env
   ```

3. Start a service:
   ```bash
   docker compose up -d
   ```

## Hardware

- **Raspberry Pi 5** — 16GB RAM, NVMe storage
- **DigitalOcean Droplet** — Basic Premium Intel (1 vCPU, 1 GB RAM, 35 GB SSD, 1 TB transfer)
- **MikroTik hEX S** router (RouterOS, VLAN segmentation)

## License

This is documentation of my personal setup — feel free to use the compose files as a reference.
