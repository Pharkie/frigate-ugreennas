# Frigate NVR on Docker

AI-powered camera monitoring with Frigate NVR, integrated with Home Assistant for smart notifications.

Should work on any Docker host, but developed and tested specifically on Ugreen NAS DXP4800+ (see [UGOS Setup Notes](docs/UGOS-SETUP.md) for NAS-specific quirks).

## Requirements

- **Host**: Any system running Docker (NAS, server, Raspberry Pi, etc.)
- **Cameras**: Any Frigate-compatible cameras (tested with Reolink via NVR)
- **Detection**: CPU or Coral TPU (CPU-only config included)

## Setup

1. Copy example files and configure:
   ```bash
   cp .env.example .env
   cp docker-compose.frigate.example.yml docker-compose.frigate.yml
   cp config/config.example.yml config/config.yml
   # Edit each file with your values
   ```

2. Deploy to NAS:
   ```bash
   ssh your-user@your-nas.local "mkdir -p /volume1/docker/frigate/config"
   # Copy files to NAS (see docs/UGOS-SETUP.md for scp workaround)
   ```

3. Start Frigate:
   ```bash
   cd /volume1/docker/frigate
   docker compose up -d
   ```

## Management

```bash
docker logs frigate -f              # View logs
docker compose restart frigate      # Restart
docker compose pull && docker compose up -d  # Update
```

## Architecture

```
Cameras/NVR (RTSP/HTTP)
    ↓ streams
Frigate (your-nas-ip:5000)
    ↓ MQTT
Home Assistant
    ↓ Automations
Notifications
```

## Key URLs

- **Frigate UI**: http://your-nas-ip:5000
- **RTSP Restream**: rtsp://your-nas-ip:8554/{camera_name}
- **Home Assistant**: http://homeassistant.local:8123

## Documentation

- [UGOS Setup Notes](docs/UGOS-SETUP.md) - NAS-specific configuration
- [Reolink Camera Guide](docs/REOLINK-CAMERAS.md) - Reolink-specific stream configuration
- [Frigate Docs](https://docs.frigate.video/) - Official documentation (camera compatibility, config reference)

## Configuration Notes

**Environment variables**: Frigate only recognizes env vars prefixed with `FRIGATE_`. Use `FRIGATE_NVR_IP`, not `REOLINK_NVR_IP`.

## Security Note

Frigate web UI has no authentication. It's only accessible on LAN via port 5000. Do not expose to internet without adding auth via Traefik or similar.
