# Claude Instructions for This Repo

This file helps Claude (or other AI assistants) understand the project context when working with this codebase. It contains general patterns and gotchas - no personal credentials or specific IPs.

---

## Documentation Strategy

- **Public docs** (tracked): Generic instructions with placeholders
- **Private config** (`.claude/config.local.md`, gitignored): Actual hostnames, IPs, usernames
- **Credentials** (`.env`, gitignored): Passwords and tokens

## Project Overview

Frigate NVR setup on Ugreen NAS DXP4800+ for AI-powered camera monitoring with Home Assistant integration.

## NAS Access

### Enabling SSH on UGOS
UGOS Control Panel → Terminal → SSH → Enable, set "Shut down automatically" to 2h.

### Docker Group
Add your user to docker group to avoid sudo for docker commands.

### scp Workaround
scp fails on UGOS. Use pipe method instead:
```bash
cat file.yml | ssh user@nas "cat > /volume1/docker/frigate/file.yml"
```

### Running commands with sudo
```bash
ssh user@nas "echo 'PASS' | sudo -S <command>"
```

## UGOS Learnings

### What Works
- Docker with NET_ADMIN capability
- Static IP assignment for containers
- Volume mounts from /volume1/
- Docker Compose v2
- Traefik reverse proxy with Cloudflare SSL

### What Doesn't Work / Gotchas
- **nginx owns ports 80/443**: Use alternate ports for Traefik
- **scp fails**: Use cat pipe method above
- **No GPU/hardware acceleration**: DXP4800+ has no iGPU - CPU-only detection or Coral TPU
- **UGOS auto-resets nginx on reboot**: Don't fight it, use alternate ports

## Home Assistant Access

### SSH Command
```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519_ha hassio@homeassistant.local
```

### Key Directories
- `/config/` - HA configuration
- `/config/.storage/` - UI-configured entities

### HA CLI
```bash
ha core restart       # Restart HA Core
ha addons list        # List installed add-ons
ha backups list       # List backups
```

### Editing HA Config Files via SSH
HA config files are owned by root. The hassio user MUST use sudo to edit:
```bash
# Use password auth (key may not work)
sshpass -p 'YOUR_SSH_PASSWORD' ssh hassio@homeassistant.local "sudo sed -i 's|old|new|g' /homeassistant/automations.yaml"

# Reload after editing
source .env && curl -X POST "$HA_URL/api/services/automation/reload" -H "Authorization: Bearer $HA_TOKEN"
```
NOTE: scp doesn't work on HA SSH add-on. Use sed -i or cat pipe with sudo.

### HA REST API (preferred)
Use API instead of SSH when possible. Token stored in `.env` (gitignored).
```bash
source .env
curl -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/states"
```

### Config Approach
User prefers **UI-editable config** where possible.

## Reolink Cameras

### What Works
- **RTSP streams** - reliable
- **HA Reolink integration** - use API for camera info

### What Doesn't Work
- **HTTP-FLV** - RLN8-410 doesn't support FLV pull streams

### Stream Configuration
- Use RTSP (HTTP-FLV not supported on RLN8-410)
- RTSP channel = NVR channel + 1 (e.g., NVR channel 1 → `h264Preview_02_main`)

### Environment Variable Naming
Frigate only recognizes env vars prefixed with `FRIGATE_`. Use `FRIGATE_NVR_IP`, not `REOLINK_NVR_IP`.

## Frigate

### Paths
- **NAS Path**: `/volume1/docker/frigate/`
- **Recordings**: `/volume1/Media/frigate/`

### Ports
- 5000: Web UI (no auth)
- 8554: RTSP restream
- 8555: WebRTC

### Management Commands (on NAS)
```bash
cd /volume1/docker/frigate
docker logs frigate -f          # View logs
docker compose restart frigate  # Restart
docker compose pull && docker compose up -d  # Update
```

### HA Integration
Install via HACS UI:
1. HACS → Integrations → Search "Frigate" → Download
2. Restart Home Assistant
3. Settings → Devices & Services → Add Integration → Frigate
4. Enter Frigate URL (NAS IP + port 5000)

## On Auth Failures
On ANY SSH or sudo authentication failure, immediately ask the user for the correct password. Don't retry or debug - just ask.

## IMPORTANT: Always Check .env First
Before asking the user for credentials, ALWAYS check `.env` in the project root. It contains:
- `HA_TOKEN` - Home Assistant long-lived access token
- `HA_URL` - Home Assistant URL
- Other credentials as needed

Also check `.claude/config.local.md` for SSH passwords and hostnames.

## HA API Usage
Always prefer the HA REST API over SSH for Home Assistant operations:
```bash
source .env
curl -X POST "$HA_URL/api/services/automation/reload" -H "Authorization: Bearer $HA_TOKEN"
```
