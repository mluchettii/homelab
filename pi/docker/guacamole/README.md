# Apache Guacamole — Remote Desktop Gateway

Clientless remote desktop gateway supporting VNC, RDP, and SSH via HTML5. Runs three containers:

- **guacd** — protocol proxy (VNC/RDP/SSH)
- **postgres** — user/connection metadata store
- **guacamole** — web application (served behind a reverse proxy)

## Setup

1. Edit `.env` with your database credentials and desired port.

2. Start the stack:
   ```bash
   docker compose up -d
   ```

3. Access the web interface at `http://<pi-ip>:8040` (or your configured port).

## SSO with Authentik

Enable OpenID SSO by setting the following in `.env`:

```bash
GUAC_OPENID_ENABLED=true
GUAC_OPENID_AUTH_ENDPOINT=https://auth.example.com/oauth/authorize
GUAC_OPENID_CLIENT_ID=guacamole
GUAC_OPENID_ISSUER=https://auth.example.com
GUAC_OPENID_JWKS=https://auth.example.com/application/oidc/jwks/
GUAC_OPENID_REDIRECT=https://guacamole.example.com/guacamole/
```

## Features

- **Drive directory** — persistent file storage (`./drive`)
- **Recording directory** — session recordings (`./record`)
- **PostgreSQL data** — persistent database (`./pgdata`)
- **Custom init scripts** — place SQL in `./init/` for database initialization

### Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GUAC_DB_USER` | `guacadmin` | PostgreSQL user |
| `GUAC_DB_PASSWORD` | *(required)* | PostgreSQL password |
| `GUAC_WEB_PORT` | `8040` | Web interface port |
| `GUAC_DRIVE_DIR` | `./drive` | Drive storage directory |
| `GUAC_RECORD_DIR` | `./record` | Recording storage directory |
| `GUAC_PGDATA_DIR` | `./pgdata` | PostgreSQL data directory |

## Architecture

```
guacd (protocol proxy) ←→ guacamole (web app) ←→ PostgreSQL (metadata)
     ↕                         ↕
  VNC/RDP/SSH clients    Reverse proxy (Nginx)
```

The `guacd` and `guacamole` containers communicate over the `guacnetwork` Docker network. The embedded nginx service is disabled — use the repo's Nginx Proxy Manager as the reverse proxy instead.
