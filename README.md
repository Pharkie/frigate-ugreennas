# Frigate NVR on Ugreen NAS

AI-powered camera monitoring with Frigate NVR on Ugreen NAS DXP4800+, integrated with Home Assistant for smart notifications.

## Hardware

- **NAS**: Ugreen DXP4800+ (no iGPU - CPU-only detection)
- **Cameras**: Reolink cameras via NVR
- **Detection**: CPU (4 threads) - Coral TPU optional upgrade

## Quick Start

```bash
# SSH to NAS
ssh your-user@your-nas.local

# Navigate to Frigate
cd /volume1/docker/frigate

# View logs
docker logs frigate -f

# Restart
docker compose restart frigate

# Update
docker compose pull && docker compose up -d
```

## Architecture

```
Reolink NVR (your-nvr-ip)
    ↓ RTSP streams
Frigate (your-nas-ip:5000)
    ↓ MQTT
Home Assistant (your-ha-ip)
    ↓ Automations
Notifications
```

## Key URLs

- **Frigate UI**: http://your-nas-ip:5000
- **RTSP Restream**: rtsp://your-nas-ip:8554/{camera_name}
- **Home Assistant**: http://homeassistant.local:8123

## Documentation

- [UGOS Setup Notes](docs/UGOS-SETUP.md) - NAS-specific configuration
- [Reolink Camera Guide](docs/REOLINK-CAMERAS.md) - Stream types and troubleshooting
- [Frigate Docs](https://docs.frigate.video/) - Official documentation

## Configuration Notes

**Environment variables**: Frigate only recognizes env vars prefixed with `FRIGATE_`. Use `FRIGATE_NVR_IP`, not `REOLINK_NVR_IP`.

## Security Note

Frigate web UI has no authentication. It's only accessible on LAN via port 5000. Do not expose to internet without adding auth via Traefik or similar.
