# Tenable Nessus

Self-hosted vulnerability scanner for detecting software flaws, missing patches, misconfigurations, default credentials, and insecure protocols.

## Access

- **URL**: `https://nessus.ts.domain.com` (via Tailscale)
- **Port**: 8834 (host network mode)

## Deployment

```bash
cd pi/docker/nessus
docker compose up -d
```

## Initial Setup

1. Access the web console at `https://<pi-ip>:8834`
2. Wait for Nessus to initialize
3. Select a license tier:
   - **Essentials** — free, 16 hosts
   - **Professional** — trial, unlimited scanning
   - **Expert** — trial, cloud + attack surface scanning
4. Create admin credentials
5. Wait for plugin downloads to complete

## Configuration

Nessus runs in host network mode, so it binds directly to the Pi's network interfaces. The Docker socket is mounted read-only for container-aware scanning.

## Scanning

Create scans targeting your homelab subnets:
- `10.0.0.0/24` — LAN (home devices)
- `10.0.30.0/24` — Pi VLAN (services)
- `100.64.0.0/10` — Tailscale tailnet

## Notes

- Nessus Essentials has a 16-host limit per scanner
- Plugin updates can take significant time on first run
- Full scans of the homelab typically complete in 15–30 minutes
