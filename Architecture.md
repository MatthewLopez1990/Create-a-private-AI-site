# Raspberry Pi WebUI Server – Infrastructure Documentation

**Documentation Type:** Template & Reference Guide  
**Last Updated:** December 09, 2025

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start-tldr)
3. [Executive Summary](#executive-summary)
4. [Network Architecture](#network-architecture)
5. [Component Details](#component-details)
6. [Docker-Compose Configuration](#docker-compose-configuration)
7. [File Tree](#file-tree-configuration-files-structure)
8. [Security Considerations](#security-considerations)
9. [Installation Steps](#installation-steps)
10. [Implementation Checklist](#implementation-checklist)
11. [Maintenance Notes](#maintenance-notes)
12. [Troubleshooting](#troubleshooting)
13. [Advanced Configuration](#advanced-configuration)
14. [Additional Resources](#additional-resources)

---

## Prerequisites

Before setting up this infrastructure, ensure you have:

- Raspberry Pi 4 or 5 (4GB+ RAM recommended)
- Ubuntu Server or Raspberry Pi OS installed
- Docker and Docker Compose installed
- A domain name (DuckDNS, NoIP, or similar free service)
- Groq API key (free tier available at https://groq.com)
- Basic familiarity with Docker and command-line tools

---

## Quick Start (TL;DR)

For experienced users who want to get started immediately:

```bash
# 1. Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 2. Create directories
mkdir -p ~/mcp-{data,projects,shared} ~/searxng_data

# 3. Create docker-compose.yml and Caddyfile (see configuration sections below)
# 4. Replace {your-domain} and {your-groq-api-key} with actual values

# 5. Start services
cd ~
docker compose up -d

# 6. Access at https://{your-domain}
```

For detailed step-by-step instructions, continue reading below.

---

## Executive Summary

This Raspberry Pi hosts a small but fully-featured AI-assistant platform.

- **Open WebUI** (v0.6.41) provides the chat UI; it runs inside a Docker container.
- **SearXNG** supplies privacy-preserving web-search capabilities, also containerised.
- **Caddy** acts as a reverse-proxy, terminating TLS for your public domain and routing traffic to the appropriate backend services.
- **Groq** is the external AI-model provider; the Open WebUI container calls the Groq API using an API key stored in an environment variable.
- **MCP Filesystem Server** (exposed on port 8766) gives the Open WebUI container filesystem access for reading/writing user data.

All components are orchestrated with a single `docker-compose.yml` file located in your home directory.

> **Note:** Throughout this documentation, replace:
> - `{username}` with your actual Linux username
> - `{your-domain}` with your actual domain name (e.g., `mypi.duckdns.org`)
> - `{your-groq-api-key}` with your Groq API key

---

## Network Architecture

```
+-------------------+      HTTPS (443)       +--------+      HTTP (80)      +-----------+
| External Client  | ---------------------> | Caddy  | ------------------> | OpenWebUI |
+-------------------+                        +--------+                     +-----------+
                                                               |
                                                               | HTTP (internal)
                                                               v
                                                          +-----------+
                                                          | SearXNG   |
                                                          +-----------+

OpenWebUI --> Groq API (HTTPS, external)
OpenWebUI --> MCP Filesystem Server (http://localhost:8766)
```

---

## Component Details

### 1. Open WebUI

- **Version:** 0.6.41
- **Container name:** *(defined in `docker-compose.yml` – not visible here)*
- **Ports:** Typically `8080` inside the container, exposed via Caddy (HTTPS 443)
- **Volumes mounted:**
  - `./mcp-shared:/app/shared` (shared folder)
  - `./mcp-data:/app/data` (persistent data)
- **Environment variables (names only):**
  - `WEBUI_HOST`
  - `WEBUI_PORT`
  - `GROQ_API_KEY` (value hidden)
  - `SEARXNG_URL` (points to the SearXNG service)
- **Key configuration:** Uses the MCP filesystem server for file access; forwards search queries to SearXNG; calls Groq for model inference.

### 2. SearXNG

- **Container name:** *(defined in `docker-compose.yml`)*
- **Ports:** Usually `8080` inside the container, reachable only from the internal network (Caddy does not expose it directly).
- **Configuration file location:** `/home/{username}/searxng_data/settings.yml`
- **Key settings (typical):**
  - `search` → enabled engines (e.g., DuckDuckGo, Bing)
  - `privacy` → `referrer_policy`, `use_http` disabled, `safe_search` enabled
- **Privacy settings:** No telemetry, `HTTP_REFERER` stripped, optional Tor support.

### 3. Caddy

- **Configuration file location:** `/home/{username}/Caddyfile`
- **Domains configured:** `{your-domain}` (e.g., `mypi.duckdns.org`)
- **SSL/TLS settings:** Automatic HTTPS via Let's Encrypt, TLS-1.3 enforced.
- **Reverse-proxy routes:**
  - `/` → `http://open-webui:8080` (Open WebUI)
  - `/search/` → `http://searxng:8080` (SearXNG)
- **Security headers:** `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`.

### 4. Groq Integration

- **How connected:** Open WebUI container makes HTTPS calls to `https://api.groq.com` using the API key stored in the environment variable `GROQ_API_KEY`.
- **Models available:** `mixtral-8x7b-32768`, `llama3-70b-8192`, etc. (as defined by the Groq subscription).
- **API configuration location:** Inside the Open WebUI environment (no separate file).

### 5. MCP Filesystem Server

- **Service file location:** Systemd unit `mcpo.service` (typically under `/etc/systemd/system/`)
- **Port:** `8766` (exposed via the `mcpo` proxy)
- **Allowed directories:** Home directory (`/home/{username}/`) and sub-folders listed in `mcp-config.json`
- **Proxy (mcpo) configuration:** Handles authentication (token-based) and maps filesystem calls to the Docker containers

---

## Docker-Compose Configuration

Create this file at `/home/{username}/docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:0.6.41
    restart: unless-stopped
    ports: [ "8080:8080" ]          # internal only; Caddy proxies HTTPS
    volumes:
      - ./mcp-data:/app/data
      - ./mcp-shared:/app/shared
    environment:
      - WEBUI_HOST=0.0.0.0
      - WEBUI_PORT=8080
      - GROQ_API_KEY={your-groq-api-key}  # Replace with your actual Groq API key
      - SEARXNG_URL=http://searxng:8080

  searxng:
    image: searxng/searxng:latest
    restart: unless-stopped
    volumes:
      - ./searxng_data:/etc/searxng
    environment:
      - BASE_URL=/search
    # No ports exposed externally; only reachable via Caddy

  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports: [ "80:80", "443:443" ]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

### Caddyfile Configuration

Create this file at `/home/{username}/Caddyfile`:

```
{your-domain} {
    reverse_proxy open-webui:8080
}

{your-domain}/search/* {
    reverse_proxy searxng:8080
}
```

---

## File Tree (configuration files structure)

```
/home/{username}
├─ Caddyfile                     # Caddy reverse-proxy config
├─ docker-compose.yml            # Docker-Compose orchestration
├─ get-docker.sh                 # Helper script (optional)
├─ mcp-config.json               # MCP server settings
├─ mcp-config.json.bak           # Backup configuration
├─ mcp-data/                     # MCP persistent data
├─ mcp-projects/                 # MCP project files
├─ mcp-shared/                   # Shared folder for documentation
├─ open-webui-setup.md           # Open WebUI setup notes (optional)
├─ searxng_data/                 # SearXNG config & data (settings.yml inside)
└─ test_server/                  # Test files (optional)
```

### Required Directory Setup

Before starting, create the necessary directories:

```bash
mkdir -p ~/mcp-data ~/mcp-projects ~/mcp-shared ~/searxng_data
```

---

## Security Considerations

| Aspect | Observation |
|--------|-------------|
| **External exposure** | Only Caddy is reachable from the internet (HTTPS 443). All other services are bound to the internal Docker network. |
| **Authentication** | - Caddy can be configured with HTTP basic auth or OAuth. <br> - Open WebUI requires user registration/login (configure on first use). <br> - Groq API key is stored in an environment variable. |
| **Recommended hardening** | - Ensure Caddyfile uses Let's Encrypt for production (automatic with valid domain). <br> - Verify that SEARXNG_URL is only reachable from Open WebUI container. <br> - Restrict MCP server to the Docker network (`127.0.0.1:8766`). <br> - Rotate Groq API keys periodically. <br> - Use strong passwords for Open WebUI accounts. <br> - Consider enabling firewall rules (ufw) to restrict access. |
| **File permissions** | Ensure proper ownership: `chown -R {username}:{username} /home/{username}/mcp-*` |

---

## Maintenance Notes

### Restart services

```bash
cd /home/{username}
docker compose restart

# or for a complete restart
docker compose down && docker compose up -d
```

### View logs

```bash
# View all container logs
docker compose logs -f

# View specific container logs
docker logs open-webui-1
docker logs searxng-1
docker logs caddy-1
```

### Log locations

- **Docker logs:** `docker logs <container_name>`
- **Caddy logs:** inside the Caddy container (`/data/logs` if configured)
- **MCP logs:** typically under `/home/{username}/mcp-data/logs`

### Backup recommendations

```bash
# Create timestamped backup
cd /home/{username}
tar -czf backup-$(date +%Y%m%d).tar.gz mcp-data/ mcp-projects/ mcp-shared/ searxng_data/ docker-compose.yml Caddyfile mcp-config.json
```

- Schedule regular backups using cron
- Store backups off-device (USB drive, network storage, cloud)
- Test restore procedures periodically
- Keep versioned copies of `docker-compose.yml` and `Caddyfile`

---

## Installation Steps

### 1. Install Docker and Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker {username}

# Log out and back in for group changes to take effect
```

### 2. Set up directory structure

```bash
cd ~
mkdir -p mcp-data mcp-projects mcp-shared searxng_data
```

### 3. Create configuration files

Create the following files in your home directory:
- `docker-compose.yml` (see Docker-Compose Configuration section)
- `Caddyfile` (see Caddyfile Configuration section)
- `mcp-config.json` (MCP server settings - see MCP documentation)

### 4. Configure your domain

- Set up DuckDNS or similar dynamic DNS service
- Point your domain to your Raspberry Pi's public IP
- Update the `{your-domain}` placeholder in Caddyfile

### 5. Set environment variables

```bash
# Add to ~/.bashrc or create a .env file
export GROQ_API_KEY="{your-groq-api-key}"
```

### 6. Start the services

```bash
cd ~
docker compose up -d
```

### 7. Verify deployment

```bash
# Check all containers are running
docker compose ps

# Check logs
docker compose logs -f
```

### 8. Access your instance

- Navigate to `https://{your-domain}` in your browser
- Complete Open WebUI initial setup
- Create your admin account

---

## Implementation Checklist

| Task | Status | Notes |
|------|--------|-------|
| Docker installed | ☐ | `docker --version` to verify |
| Directories created | ☐ | `mcp-data`, `mcp-projects`, `mcp-shared`, `searxng_data` |
| `docker-compose.yml` created | ☐ | Update GROQ_API_KEY |
| `Caddyfile` created | ☐ | Update domain name |
| Domain configured | ☐ | DuckDNS or similar |
| Groq API key obtained | ☐ | From https://groq.com |
| MCP config created | ☐ | `mcp-config.json` |
| Services started | ☐ | `docker compose up -d` |
| HTTPS working | ☐ | Let's Encrypt auto-configured |
| Open WebUI accessible | ☐ | Admin account created |
| SearXNG functional | ☐ | Test search queries |

---

## Troubleshooting

### Common Issues

**Containers won't start:**
```bash
# Check logs for errors
docker compose logs

# Ensure ports aren't already in use
sudo netstat -tulpn | grep -E ':(80|443|8080|8766)'
```

**Can't access via domain:**
- Verify DNS propagation: `nslookup {your-domain}`
- Check Caddy logs: `docker logs caddy-1`
- Ensure ports 80 and 443 are forwarded in your router
- Verify Raspberry Pi has a static local IP

**Let's Encrypt certificate issues:**
- Ensure domain is publicly accessible
- Check Caddy logs for ACME errors
- Verify email is set in Caddyfile (if required)

**SearXNG not returning results:**
- Check SearXNG logs: `docker logs searxng-1`
- Verify settings.yml configuration
- Test direct access (if exposed): `curl http://localhost:8080/search`

---

## Advanced Configuration

### Custom Models

You can add other model providers by updating the Open WebUI environment variables:

```yaml
environment:
  - OPENAI_API_KEY={your-openai-key}
  - ANTHROPIC_API_KEY={your-anthropic-key}
```

### Performance Tuning

For Raspberry Pi 4 (4GB RAM):
- Consider limiting Open WebUI workers
- Monitor with: `docker stats`
- Adjust Docker memory limits if needed

### Additional Services

This stack can be extended with:
- **Postgres/MySQL** for persistent Open WebUI database
- **Redis** for caching
- **Nginx Proxy Manager** alternative to Caddy
- **Watchtower** for automatic container updates

---

## Additional Resources

- [Open WebUI Documentation](https://docs.openwebui.com/)
- [SearXNG Documentation](https://docs.searxng.org/)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Groq API Documentation](https://console.groq.com/docs)
- [MCP Documentation](https://modelcontextprotocol.io/)
- [Docker Compose Reference](https://docs.docker.com/compose/)

---

## Contributing

If you improve this setup or find issues, consider:
- Documenting your changes
- Sharing optimizations for Raspberry Pi performance
- Creating backup/restore scripts
- Adding monitoring solutions

---

## License & Credits

This documentation template is provided as-is for community use. Components used:
- Open WebUI (MIT License)
- SearXNG (AGPLv3)
- Caddy (Apache 2.0)
- Groq (API Service)

Always review individual project licenses before deployment.
