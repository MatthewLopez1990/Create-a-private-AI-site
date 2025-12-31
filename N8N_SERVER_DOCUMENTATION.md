# N8N Workflow Automation Documentation example

**Server:** Raspberry Pi (98.11.111.111)
**URL:** https://usernamen8nserver.duckdns.org
**Internal Port:** 5678
**Container Name:** `n8n`

---

## 1. Overview

N8N (Nodemation) is a workflow automation tool that allows you to connect different services and APIs together. It runs in a Docker container on your Raspberry Pi and is accessible securely via HTTPS through the Caddy reverse proxy.

## 2. Configuration Details

### Docker Container
- **Image:** `n8nio/n8n:latest`
- **Current Version:** 2.1.4
- **Restart Policy:** `unless-stopped`
- **Network:** `name_webui-network` (Shared with Open WebUI and SearXNG)
- **Internal IP:** `172.19.0.4`

### Storage (Persistence)
- **Data Volume:** `n8n_data` (Docker Volume)
- **Mount Point:** `/home/node/.n8n`
- **Purpose:** Stores all your workflows, credentials, and execution history.
- **Backup Note:** This data lives in `/var/lib/docker/volumes/n8n_data/_data` on the host. It is **NOT** included in the standard file-based backup and must be backed up separately.

### Environment Variables example
The container is configured with the following critical environment variables:

| Variable | Value | Description |
|:---|:---|:---|
| `WEBHOOK_URL` | `https://usernamen8nserver.duckdns.org` | Public URL for webhooks to reach your server. |
| `GENERIC_TIMEZONE` | `America/New_York` | Timezone for scheduled workflows. |
| `N8N_SECURE_COOKIE` | `false` | Set to false because SSL is terminated at Caddy, not N8N itself. |
| `EXECUTIONS_TIMEOUT` | `3600` | Workflows will timeout after 1 hour (3600s). |
| `N8N_WEBHOOK_TIMEOUT` | `3600` | Webhook requests will timeout after 1 hour. |

## 3. Network Architecture

```
┌─────────────────────────────┐
│      Internet (HTTPS)       │
└──────────────┬──────────────┘
               │
               ▼
      ┌─────────────────┐
      │  Caddy Proxy    │ (Host Port 443)
      │  SSL / TLS      │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │  N8N Container  │ (Port 5678)
      │  (Internal IP)  │
      └─────────────────┘
```

## 4. Operational Commands

### Check Status
```bash
ssh name@98.11.111.111 "docker ps --filter name=n8n"
```

### View Logs
```bash
ssh name@98.11.111.111 "docker logs --tail 50 n8n"
```

### Restart Service
```bash
ssh name@98.11.111.111 "docker restart n8n"
```

### Manual Backup (Critical Data)
To backup your workflows and credentials to a tar file in your home directory:

```bash
ssh name@98.11.111.111 "docker run --rm -v n8n_data:/data -v /home/<name>:/backup alpine tar czf /backup/n8n-data-backup-$(date +%F).tar.gz -C /data ."
```

## 5. Troubleshooting

- **"Connection Lost" in UI:** Usually happens if the WebSocket connection drops. Refresh the page.
- **Webhook 404:** Ensure the `WEBHOOK_URL` matches your external domain exactly.
- **Timeouts:** Long-running workflows may hit the 1-hour limit (`EXECUTIONS_TIMEOUT`). Increase this env var if needed.

---
*Documentation generated based on live system inspection on December 28, 2025.*
