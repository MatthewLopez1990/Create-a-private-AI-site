# Raspberry Pi Restore Point Documentation example

**Created:** December 25, 2025
**Server:** Raspberry Pi? - 98.11.111.111 (public) / 10.0.0.111 (local)
**Backup File:** `pi-restore-backup-202xxxxx_143946.tar.gz`

---

## What is a Restore Point?

A restore point is a snapshot of your Raspberry Pi server's configuration at a specific point in time. This backup contains all the critical information needed to recreate your server's setup if anything gets broken or lost.

---

## Backup Locations

This restore point is stored in **TWO locations** for redundancy:

1. **On the Raspberry Pi Server example:**
   - Location: `~/pi-restore-backup-202xxxxx_143946.tar.gz`
   - User: `name`
   - Accessible via: `ssh name@98.11.111.111

2. **On Your WSL (Local Computer):**
   - Location: `/home/<Name>/pi-restore-backup-202xxxxx_143946.tar.gz`
   - Size: 14 KB
   - Safe from server failures

---

## What's Included in This Backup

### 1. Docker Configuration
- **Container list**: Running containers with their status
- **Container images**: Images used by each container
- **Port mappings**: How services are exposed
- **Environment variables**: Configuration for each container
- **Networks and volumes**: Docker networking and storage configuration

**Backed up containers:**
- **Open WebUI**: `ghcr.io/open-webui/open-webui:main` (Port 3000)
- **N8N**: `n8nio/n8n:latest` (Port 5678)
- **SearXNG**: `searxng/searxng:latest` (Port 8888)

### 2. Caddy Reverse Proxy
- Complete Caddy configuration directory
- Caddyfile (main configuration file)
- SSL/TLS certificates
- Site-specific configurations for:
  - `usernameprivetai.duckdns.org`
  - `matthewn8nserver.duckdns.org`

### 3. Resume Site
- **server.py**: Main resume site application
- **requirements.txt**: Python dependencies
- Process information (PID, working directory)
- Virtual environment structure

### 4. System Configuration
- **SSH configuration**: `sshd_config`
- **Cron jobs**: Scheduled tasks
- **System services**: Running services status
- **Network configuration**: IP addresses, routing
- **System information**: OS version, disk usage, memory usage, uptime
- **Installed packages**: List of installed Debian packages

### 5. Docker Compose & Dockerfiles
- Locations of any docker-compose files
- Locations of Dockerfiles for custom builds

---

## What's NOT Included

**Important limitations of this backup:**

1. **Docker Volumes Data**: This backup does NOT include the data stored in Docker volumes. You should separately backup:
   - Open WebUI chat history and configurations
   - N8N workflows and settings
   - SearXNG settings and cache

2. **User Data**: Personal files not related to system configuration

3. **External Services**: Data stored outside the server (e.g., Duck DNS settings)

4. **Dynamic Data**: Runtime data that changes frequently

---

## How to Use This Restore Point

### Scenario 1: Need to Rebuild the Server from Scratch

If your Raspberry Pi server becomes completely corrupted or you need to move to new hardware:

1. **Extract the backup example:**
   ```bash
   cd ~
   tar -xzf pi-restore-backup-202xxxxx_143946.tar.gz
   cd pi-restore-backup
   ```

2. **Review the manifest:**
   ```bash
   cat BACKUP_MANIFEST.txt
   ```

3. **Restore Docker containers:**
   ```bash
   # Pull the images
   docker pull ghcr.io/open-webui/open-webui:main
   docker pull n8nio/n8n:latest
   docker pull searxng/searxng:latest

   # Recreate containers (check docker-commands.txt for full configuration)
   docker run -d --name open-webui -p 3000:8080 ghcr.io/open-webui/open-webui:main
   docker run -d --name n8n -p 5678:5678 n8nio/n8n:latest
   docker run -d --name searxng -p 8888:8080 searxng/searxng:latest
   ```

4. **Restore Caddy:**
   ```bash
   sudo cp caddy-config-backup/Caddyfile /etc/caddy/Caddyfile
   sudo caddy validate --config /etc/caddy/Caddyfile
   sudo systemctl reload caddy
   ```


### Scenario 2: Configuration Broke, Server Still Running

If a specific service configuration got corrupted:

1. **Check the specific configuration file:**
   ```bash
   # For Caddy
   diff /etc/caddy/Caddyfile ~/pi-restore-backup/caddy-config-backup/Caddyfile


2. **Restore the corrupted file:**
   ```bash
   sudo cp ~/pi-restore-backup/caddy-config-backup/Caddyfile /etc/caddy/Caddyfile
   sudo systemctl reload caddy
   ```

### Scenario 3: Need to Check Original Configuration

If you just need to reference the original configuration:

```bash
# Extract to a temporary location
mkdir /tmp/restore-restore
tar -xzf pi-restore-backup-202xxxxx_143946.tar.gz -C /tmp/restore-restore

# View specific files
cat /tmp/restore-restore/pi-restore-backup/docker-backup.txt
cat /tmp/restore-restore/pi-restore-backup/docker-commands.txt
```

---

## Verification After Restoration

After restoring, verify all services are running correctly:

```bash
# Check Docker containers
docker ps

# Check Caddy status
systemctl status caddy

# Check Resume Site
ps aux | grep server.py

# Test all services locally
curl -I http://localhost:3000  # Open WebUI
curl -I http://localhost:5678  # N8N
curl -I http://localhost:8888  # SearXNG

# Test external access example (if DNS is configured)
curl -I https://usernamepriveteai.duckdns.org
curl -I https://username8nserver.duckdns.org
```

---

## Backup Contents Reference

The backup archive contains these files:

| File | Description |
|------|-------------|
| `BACKUP_MANIFEST.txt` | Overview of backup contents and services |
| `docker-backup.txt` | Docker containers, images, networks, volumes |
| `docker-commands.txt` | Container environment variables and port mappings |
| `docker-compose-files.txt` | Locations of docker-compose files |
| `caddy-config-backup/` | Complete Caddy configuration directory |
| `sshd_config` | SSH daemon configuration |
| `cron-backup.txt` | User cron jobs |
| `services.txt` | Running system services status |
| `network.txt` | Network interface and routing configuration |
| `system-info.txt` | System information (OS, disk, memory, uptime) |
| `packages.txt` | List of installed packages |
| `home-config.txt` | Important files in home directory |

---

## Important Notes

1. **Keep Both Copies Safe**: This backup exists in two locations for redundancy. Do not delete the copy from the Raspberry Pi if you're still using that server.

2. **Docker Volumes Need Separate Backup**: The data in Docker volumes (chat history, workflows, settings) is NOT included in this backup. Set up separate backups for:
   ```bash
   # Example: Backup Open WebUI data
   docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine \
     tar czf /backup/open-webui-data.tar.gz -C /data .
   ```

3. **Create New Restore Points Regularly**: Create a new restore point after making significant configuration changes:
   - Adding new services
   - Changing port configurations
   - Updating Docker containers
   - Modifying Caddy configuration

4. **Test Your Backup**: Periodically test the restore process to ensure the backup is complete and usable.

5. **Document Changes**: Keep a log of configuration changes to help identify when a new restore point is needed.

---

## Quick Reference Commands

### To extract and view the backup example:
```bash
tar -xzf pi-restore-backup-202xxxxx_143946.tar.gz
cd pi-restore-backup
cat BACKUP_MANIFEST.txt
```

### To view Docker configurations:
```bash
cat docker-backup.txt
cat docker-commands.txt
```

### To view system configurations:
```bash
cat system-info.txt
cat network.txt
cat services.txt
```

### To view Caddy configuration:
```bash
cat caddy-config-backup/Caddyfile
```

---

## Support

If you encounter issues during restoration:

1. **Check service logs:**
   ```bash
   docker logs <container_name>
   journalctl -u <service_name>
   ```

2. **Verify port availability:**
   ```bash
   sudo netstat -tlnp | grep <port>
   ```

3. **Check Docker status:**
   ```bash
   docker ps
   docker network ls
   docker volume ls
   ```

4. **Review the backup contents** to ensure all necessary files were captured.

---

## Summary

This restore point captures the complete configuration of your Raspberry Pi server as of December 25, 2025. It includes all Docker containers, Caddy reverse proxy configuration, resume site setup, and system settings. The backup is stored in two locations for redundancy.

**Remember**: This backup does NOT include Docker volume data. You must separately backup persistent data from your services to ensure complete restoration.

