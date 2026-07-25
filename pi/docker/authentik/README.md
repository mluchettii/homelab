# Authentik — Identity Provider

Open-source Identity Provider with reverse proxy, SSO, and MFA. Runs three containers:

- **postgresql** — user/application data store
- **server** — web application and API
- **worker** — background tasks and outposts management

## Setup

1. Generate secrets before first run:
   ```bash
   PG_PASS=$(openssl rand -base64 36 | tr -d '\n')
   AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')
   ```

2. Edit `.env` with your generated secrets and desired port configuration.

3. Start the stack:
   ```bash
   docker compose up -d
   ```

4. Navigate to `http://authentik.ts.domain.com` to create the initial `akadmin` user.

## SSO Integration

Authentik uses OpenID Connect for SSO. Configure Outposts and providers to enable SSO for other services:

- **Guacamole** — configure OpenID in `pi/docker/guacamole/.env`
- **Vaultwarden** — configure OIDC provider in Vaultwarden admin panel

### Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PG_DB` | `authentik` | PostgreSQL database name |
| `PG_USER` | `authentik` | PostgreSQL user |
| `PG_PASS` | *(required)* | PostgreSQL password |
| `AUTHENTIK_SECRET_KEY` | *(required)* | Application secret key |
| `COMPOSE_PORT_HTTP` | `9000` | HTTP port |
| `COMPOSE_PORT_HTTPS` | `9443` | HTTPS port |

## Architecture

```
Authentik Server → PostgreSQL (database)
     ↓
Authentik Worker (background tasks, outposts)
     ↓
OIDC provider for: Guacamole, Vaultwarden, other services
```

All services communicate over the Docker network. The server and worker containers share the same image and are differentiated by their command (`server` vs `worker`).
