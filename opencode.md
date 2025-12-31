# OpenCode Configuration Example

**Owner:** username
**Role:** Job
**Company:** Example Company
**Last Updated:** December 25, 20xx

---

## Infrastructure Overview

### Raspberry Pi Server
- **Local IP:** 10.0.0.111
- **Public IP:** 98.11.111.111
- **SSH:** `ssh name@10.0.0.111`
- **OS:** Raspberry Pi OS (Debian-based)

### Possible Hosted Services example

| Service | URL | Port | Container | Purpose |
|---------|-----|------|-----------|---------|
| **Open WebUI** | https://usernamepriveteai.duckdns.org | 3000 | `open-webui` | Private AI chat interface (v0.6.41) |
| **SearXNG** | Internal only | 8888 | `searxng` | Meta search engine for Open WebUI |
| **N8N** | https://usernamen8nserver.duckdns.org | 5678 | `n8n` | Workflow automation |
| **Caddy** | N/A | 80/443 | Host service | Reverse proxy |

### Architecture Diagram example
```
Internet → Duck DNS → Public IP (98.11.111.111)
    ↓
Caddy Reverse Proxy (Port 80/443)
    ├→ usernamepriveteai.duckdns.org → Open WebUI (Port 3000)
    └→ usernamen8nserver.duckdns.org → N8N (Port 5678)
```

---

## Command Safety Tiers

### 🟢 Tier 1: Auto-Approved (Information Gathering)
These commands are safe and can be run without confirmation:
- `docker ps`, `docker logs <container>`, `docker inspect`
- `systemctl status <service>`
- `cat`, `ls`, `grep`, `find`, `tail`
- `curl -I <url>` (headers only)
- `netstat`, `ss -tuln`
- `df -h`, `free -h`, `uptime`

### 🟡 Tier 2: Confirm First (Operations)
These require user approval before executing:
- `docker restart <container>`
- `systemctl restart <service>`
- `docker-compose up -d`
- `git pull`, `git checkout`
- File edits to configs (show diff first)

### 🟠 Tier 3: Detailed Review (Infrastructure Changes)
These need explicit review and approval with clear explanation:
- Modifying Caddy configuration
- Changing DNS settings
- Port mapping changes
- Firewall rules (`ufw`, `iptables`)
- Network interface changes
- Docker network modifications

### 🔴 Tier 4: Never Without Explicit Request
**FORBIDDEN unless user specifically asks:**
- Stopping/removing containers
- `rm -rf` commands
- Modifying SSH configuration
- Changing user permissions
- Touching the resume site Python server (`server.py`) or its dependencies
- Any command that could break existing services

---

## Standard Workflows

### Full System Status Check
```bash
# Service health
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
systemctl is-active caddy
ps aux | grep server.py | grep -v grep

# Resource usage
df -h /
free -h
docker stats --no-stream

# Recent logs (last 50 lines)
docker logs --tail 50 open-webui
docker logs --tail 50 n8n
journalctl -u caddy --no-pager -n 50
```

### Docker Container Update
```bash
# 1. Pull latest image
docker pull <image>

# 2. Stop container
docker stop <container>

# 3. Backup current container
docker commit <container> <container>-backup-$(date +%Y%m%d)

# 4. Remove old container
docker rm <container>

# 5. Start new container with same config
docker run -d --name <container> [... same flags as before ...]

# 6. Verify
docker ps | grep <container>
docker logs <container>
```

### Caddy Configuration Change
```bash
# 1. Backup current config
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.backup.$(date +%Y%m%d_%H%M%S)

# 2. Edit config (show diff after)
sudo nano /etc/caddy/Caddyfile

# 3. Validate syntax
sudo caddy validate --config /etc/caddy/Caddyfile

# 4. Reload (not restart - keeps connections alive)
sudo systemctl reload caddy

# 5. Verify
sudo systemctl status caddy
curl -I https://mattopenai.duckdns.org
```

### Troubleshooting 502 Bad Gateway
```bash
# 1. Check if upstream service is running
docker ps | grep open-webui
systemctl is-active caddy

# 2. Check upstream health
curl -I http://localhost:3000

# 3. Check Caddy logs
journalctl -u caddy --no-pager -n 100 | grep -i error

# 4. Check container logs
docker logs --tail 100 open-webui

# 5. Check Caddy config
sudo caddy validate --config /etc/caddy/Caddyfile

# 6. If needed, restart services
docker restart open-webui
sudo systemctl reload caddy
```

### Resume Site Handling
**⚠️ HANDLE WITH CARE - This site must stay up!**
```bash
# Check if running
ps aux | grep server.py | grep -v grep

# Check logs (if redirected to file)
tail -f /path/to/resume/logs/server.log

# ONLY restart if explicitly requested
# (Usually runs via systemd or direct python command)
```

---

## Coding Preferences

### Code Standards

**Python:**
- Type hints for function signatures
- Docstrings (Google style)
- Error handling with specific exceptions
- Use `pathlib` for file paths
- Follow PEP 8

**PowerShell:**
- Comment-based help blocks
- `[CmdletBinding()]` for advanced functions
- `try/catch/finally` error handling
- Parameter validation attributes
- Follow PowerShell naming conventions (Verb-Noun)

**JavaScript/TypeScript:**
- JSDoc comments for functions
- `async/await` over promises
- Error handling with try/catch
- ESLint compliant
- Prefer `const` over `let`

**Docker/Compose:**
- Include health checks
- Use named volumes
- Document port mappings
- Environment variables for config
- Multi-stage builds when possible

---

## WSL Environment

### Current Setup
- **OS:** Windows 11 with WSL 2
- **Distribution:** Ubuntu
- **Working Directory:** Usually in `/mnt/c/Users/<name>/...`
- **Network:** Can access Windows `localhost` via WSL's IP

### Key Considerations
- File paths use `/mnt/c/` for Windows drives
- Git operations best done from WSL for line endings
- Docker Desktop integration with WSL 2
- VS Code with Remote-WSL extension

---

## OpenCode vs. Gemini CLI vs. Claude Code

### When to Use Each

**Use OpenCode When:**
- Writing or refactoring code
- Need CLI-focused workflows
- Want project-aware context
- Terminal-focused workflows
- Rapid iteration on commands

**Use Claude Code When:**
- Need Skills system for domain knowledge
- IDE integration needed
- MCP server access required
- Complex multi-step tasks

**Use Gemini CLI When:**
- Quick infrastructure tasks
- Server administration
- One-off scripts

### Integration
All tools can work together:
- Use OpenCode for development
- Use Gemini for infrastructure ops
- Use Claude Code for specialized skills
- Share context via documentation files (this file, GEMINI.md, CLAUDE.md)

---

## Emergency Procedures

### Rollback Container
```bash
# List backups
docker images | grep backup

# Stop current container
docker stop <container>

# Start backup
docker run -d --name <container> <container>-backup-<date> [... flags ...]

# Verify
docker logs <container>
```

### Restore Caddy Config
```bash
# List backups
ls -lh /etc/caddy/Caddyfile.backup.*

# Restore
sudo cp /etc/caddy/Caddyfile.backup.YYYYMMDD_HHMMSS /etc/caddy/Caddyfile

# Validate and reload
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

### Service Won't Start
```bash
# Check detailed logs
journalctl -xeu <service>

# Check config syntax
<service> -t  # (e.g., caddy validate)

# Check file permissions
ls -la /path/to/config

# Check port conflicts
sudo netstat -tlnp | grep <port>
```

---

## Quick Reference

### SSH to Pi
```bash
ssh usert@10.0.0.111
```

### One-Liner Full Status
```bash
ssh user@10.0.0.111 'docker ps && systemctl is-active caddy && ps aux | grep server.py | grep -v grep'
```

### Port Map
| Port | Service |
|------|---------|
| 22 | SSH |
| 80/443 | Caddy |
| 3000 | Open WebUI |
| 5678 | N8N |
| 8888 | SearXNG |

---

## Remember

> **Fix or modify ONLY what is specifically requested. Never break other services. When in doubt, ASK. When touching infrastructure, BACKUP first and VERIFY after.**

---

## Harvard Class Preparation

Matthew is preparing for a Harvard class on AI agent transformation in enterprises. Key topics:
- Enterprise AI integration patterns
- Agent orchestration and workflows
- Security and governance
- ROI measurement and adoption strategies

---

## Hardware Wishlist

**CanaKit Raspberry Pi 5 Starter Kit PRO** (Model B0CRSNCJ6Y)
- Amazon: http://amazon.com/CanaKit-Raspberry-Starter-Kit-PRO/dp/B0CRSNCJ6Y/
- Use case: Additional capacity for testing environment, or failover

---

*End of OPENCODE.md*