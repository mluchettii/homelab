# RustDesk

Open-source remote desktop server — a self-hosted alternative to TeamViewer and AnyDesk. Provides secure remote access with end-to-end encryption, file transfer, and chat.

## Access

- **ID/Relay Server**: `rustdesk.ts.mydomain.com` (via Tailscale)
- **Ports**: 21114–21119 (TCP), 21116 (UDP)

## Deployment

```bash
cd pi/docker/rustdesk
docker compose up -d
```

## Client Configuration

After deployment, retrieve your server's public key:

```bash
docker logs rustdesk
```

Look for the `Key:` line in the output. Then configure RustDesk clients:

1. Download the client from <https://github.com/rustdesk/rustdesk/releases>
2. Navigate to **Network → ID/Relay server**
3. Enter:
   - **ID server**: `rustdesk.ts.mydomain.com:21116`
   - **Key**: (from `docker logs rustdesk`)
   - **Relay server** and **API server** are auto-detected
4. Verify the server status shows **Ready**

## Usage

To control a remote host, enter its RustDesk ID in the **Control Remote Desktop** field and connect. Clients will prompt for the set password or one-time password.

## Notes

- Runs in host network mode for direct port binding
- The `hbbs` container generates identity keys on first run
- Client devices can connect over Tailscale without port forwarding on the router
