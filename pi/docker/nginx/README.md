# Nginx Proxy Manager + GoAccess

Reverse proxy with Let's Encrypt SSL and web traffic analytics.

## Services

### Nginx Proxy Manager

Runs in `host` network mode, binding directly to ports 80/443 on the Pi. Manages reverse proxy hosts, SSL certificates, and access lists.

- **Data directory** — `./data/npm_data/` (SQLite database, proxy config)
- **Let's Encrypt** — `./data/npm_letsencrypt/` (SSL certificates)
- **Logs** — `./data/npm_logs/` (access/error logs)

### GoAccess

Real-time web traffic analytics for Nginx Proxy Manager. Parses access logs to produce a live dashboard showing:

- Visitor count, bandwidth, and request rates
- Top requested URLs, referrers, and search engine keywords
- Operating system, browser, and device breakdown
- HTTP status codes and response times

Accessible at `nginx.ts.domain.com` via reverse proxy; GoAccess dashboard is bound to localhost only (`:7880`).

## Setup

1. Start the stack:
   ```bash
   docker compose up -d
   ```

2. Navigate to `http://<pi-ip>:81` to access the Nginx Proxy Manager admin panel. Create an initial admin account.

3. Configure proxy hosts and SSL certificates through the web UI.

## Access

- **Nginx Proxy Manager** — `nginx.ts.domain.com` (reverse proxy, HTTP/HTTPS)
- **NPM Admin Panel** — `http://<pi-ip>:81` (direct access only)
- **GoAccess** — `http://<pi-ip>:7880` (localhost only)

## Configuration Notes

- **AdGuard Home port conflict** — AdGuard Home uses port 443, so it's bound to `127.0.0.1:4433` to avoid conflicts with Nginx Proxy Manager.
- **GoAccess log path** — configured to read from `./data/npm_data/logs/` (Nginx Proxy Manager's log directory).
- **SQLite database** — Nginx Proxy Manager uses SQLite instead of MySQL for simplicity (`/data/database.sqlite`).

### Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_SQLITE_FILE` | `/data/database.sqlite` | SQLite database path |
| `TZ` | *(system)* | Timezone for GoAccess |
| `BASIC_AUTH` | `false` | Enable basic auth on GoAccess |
| `BASIC_AUTH_USERNAME` | — | GoAccess basic auth username |
| `BASIC_AUTH_PASSWORD` | — | GoAccess basic auth password |

## Architecture

```
Internet → Nginx Proxy Manager (host:80, host:443)
    ↓
Reverse proxy → Internal services (via Tailscale or LAN)
    ↓
GoAccess → Parses access logs → Dashboard (:7880)
```
