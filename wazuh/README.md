# Wazuh Security Stack

Configuration files for the Wazuh SIEM/XDR stack running on the Raspberry Pi, plus the n8n SOAR automation workflow that integrates Wazuh with local LLM alert triage and ntfy notifications.

Detailed walkthroughs are in the [zenportfolio](https://mluchetti.com/docs/security/wazuh/) (Wazuh setup) and [zenportfolio](https://mluchetti.com/docs/automation/n8n/) (n8n SOAR workflow).

## Files

### `ossec.conf`

Wazuh server configuration (`/var/ossec/etc/ossec.conf`). Includes:

- **File Integrity Monitoring (FIM)** — monitors `/home` with real-time scanning, 500K entry database limit, ignores noisy directories. FIM can be configured per-agent; the Droplet agent example uses `<ignore>` to skip noisy paths
- **VirusTotal integration** — submits file hashes from FIM events to VirusTotal API for malware detection. Requires `check_all="yes"` on monitored directories to record hashes
- **Custom n8n webhook integration** — forwards alerts with level ≥ 9 to the n8n SOAR workflow

Replace `VT_API_KEY` and `YOUR_WEBHOOK_PATH` with actual values before deploying.

### `integrations/custom-n8n`

Shell wrapper script installed at `/var/ossec/integrations/custom-n8n`. Resolves the Wazuh Python interpreter path and delegates to the accompanying `.py` script.

### `integrations/custom-n8n.py`

Python script that reads a Wazuh alert JSON file, extracts key fields (agent info, rule details, MITRE ATT&CK mapping, source IPs), and POSTs a structured payload to the n8n webhook URL. The `custom-n8n` integration in `ossec.conf` calls this script with the alert file path and webhook URL as arguments.

### `../pi/docker/automation/n8n-workflows/wazuh-hybrid-soar.json`

n8n workflow JSON — the complete SOAR automation. Two pipelines:

1. **AI triage** — high-severity alerts (level ≥ 9) are sent to a self-hosted Ollama `qwen2.5:3b` model for SOC-style investigation reports, then formatted and sent to ntfy. The prompt asks for alert name, description, MITRE ATT&CK mapping, impacted scope, reputation checks, analysis, verdict, and security recommendations
2. **Malware investigation** — FIM malware hash alerts (rule IDs 110002, 87105) are validated against VirusTotal, with results sent via ntfy (with clickable VT hash link) and formatted HTML email

### `../pi/docker/automation/ntfy/config/server.yml`

ntfy server configuration. This file belongs in your ntfy container's config directory:

```
../pi/docker/automation/ntfy/config/server.yml
```

It configures HTTPS (via Let's Encrypt certs from Nginx Proxy Manager), per-user auth with deny-all default, and reverse-proxy mode. Topics are created with separate read-only and write-only users, each with their own auth tokens.

## Architecture

```
Wazuh Agent → Wazuh Server (Pi)
    ↓ (level ≥ 9 alerts)
custom-n8n integration
    ↓ (webhook POST)
n8n SOAR workflow
    ├── AI triage → Ollama qwen2.5:3b → ntfy
    └── Malware check → VirusTotal → ntfy + Gmail
```

All communication between Wazuh, n8n, and Ollama happens over the `automation-stack` Docker network on the Pi. The ntfy webhook is exposed via Tailscale (`n8n.ts.mydomain.com`).

## Tailscale ACL

Wazuh agents communicate with the server over Tailscale — no port forwarding required. The ACL grant that allows this:

```json
{
  "src": ["tag:clients"],
  "dst": ["tag:pi"],
  "ip": ["tcp:1514", "tcp:1515", "tcp:55000"]
}
```

This is included in `tailscale/acl.json` in the repo root.

## Credits

This SOAR workflow and integration scripts are adapted from community projects:

- **[Mariskarthick M.](https://n8n.io/workflows/6978-automate-wazuh-alert-triage-and-reporting-with-gpt-4o-mini-and-telegram/)** — n8n workflow for Wazuh alert triage and reporting with GPT-4o-mini and Telegram
- **[Rajneesh G.](https://n8n.io/workflows/5997-malicious-file-detection-and-response-wazuh-to-virustotal-with-slack-alerts/)** — n8n workflow for malicious file detection and response from Wazuh to VirusTotal with Slack alerts
- **[eaglefin](https://github.com/eaglefn/wazuh-n8n-workflow)** — custom n8n webhook integration scripts (`custom-n8n` shell wrapper and `custom-n8n.py` Python script)
