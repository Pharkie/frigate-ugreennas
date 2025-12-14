# AI-Described Security Clips to Your Phone — In 80 Lines

Frigate detects objects, LLM describes the scene, and you get a video clip notification on your phone.

## Why This Exists

I struggled to understand and get working the [500+ line Blueprint](https://community.home-assistant.io/t/blueprint-frigate-vision-ai-powered-notifications-with-llm-recognition-cooldowns-multi-cam-logic-v0-9/907582/13) and its [1000+ line derivatives](https://community.home-assistant.io/t/frigate-ai-notification-multi-camera-include-exclude-zones-llmvision-low-noise/937467). So, I reverse engineered it back to ~80 lines of simple YAML: specify a camera, specify detection events (person, cat), get an AI summary, send a notification to iOS with the video clip. That's it.

## Compatibility

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
Cameras/NVR (RTSP)
    ↓
Frigate (local object detection)
    ↓ MQTT event: "person detected"
Home Assistant
    ↓ triggers automation
LLM Vision (analyzes 3 frames from clip)
    ↓
Notification (LLM description + clip link)
```

## Key URLs

- **Frigate UI**: http://your-nas-ip:5000
- **RTSP Restream**: rtsp://your-nas-ip:8554/{camera_name}
- **Home Assistant**: http://homeassistant.local:8123

## AI-Enhanced Notifications

**How it works**: Frigate runs object detection locally (no cloud). When it detects something (person, cat, etc.), Home Assistant triggers an automation that sends 3 frames from the recorded clip to an LLM (e.g., Google Gemini) for a text description. You get a notification with the LLM's description + a link to the clip.

To set this up:

1. Install [LLM Vision](https://github.com/valentinfrlch/ha-llmvision) via HACS
2. Configure your AI provider (e.g., Google Gemini API key)
3. Copy the example automation:
   ```bash
   cp config/automations.example/hallway-cat-detection.yaml config/automations/
   # Edit with your camera, provider ID, and notify service
   ```
4. Import into Home Assistant (Settings → Automations → Create → Edit as YAML)

See [config/automations.example/](config/automations.example/) for a simple, working automation that avoids the complexity and bugs of larger blueprints.

## Documentation

- [UGOS Setup Notes](docs/UGOS-SETUP.md) - NAS-specific configuration
- [Reolink Camera Guide](docs/REOLINK-CAMERAS.md) - Reolink-specific stream configuration
- [Frigate Docs](https://docs.frigate.video/) - Official documentation (camera compatibility, config reference)

## Configuration Notes

**Environment variables**: Frigate only recognizes env vars prefixed with `FRIGATE_`. Use `FRIGATE_NVR_IP`, not `REOLINK_NVR_IP`.

## Security Note

Frigate web UI has no authentication. It's only accessible on LAN via port 5000. Do not expose to internet without adding auth via Traefik or similar.
