# UGOS Setup Notes

Ugreen NAS-specific configuration notes for Frigate deployment.

## Enabling SSH

1. Open UGOS web interface
2. Go to **Control Panel → Terminal → SSH**
3. Check **Enable**
4. Set **Shut down automatically** to **2h later** (security best practice)
5. Click **Apply**

![UGOS SSH Settings](images/UGOS-SSH.png)

## SSH Connection

```bash
ssh your-user@your-nas.local
# Or by IP:
ssh your-user@your-nas-ip
```

## Docker Commands

your-user is in the docker group, so no sudo needed:

```bash
docker ps
docker compose up -d
docker logs frigate
```

## File Transfer Workaround

Standard `scp` fails on UGOS. Use this method instead:

```bash
# From local machine:
cat docker-compose.frigate.yml | ssh your-user@your-nas.local "cat > /volume1/docker/frigate/docker-compose.frigate.yml"
```

## Directory Structure

```
/volume1/
├── docker/
│   └── frigate/
│       ├── docker-compose.frigate.yml
│       └── config/
│           └── config.yml
└── Media/
    └── frigate/           # Clips and snapshots storage
        ├── clips/
        └── recordings/
```

## Network Considerations

### Existing Infrastructure
The NAS already has a Docker network from the arr-stack project:
- **Network**: traefik-proxy (192.168.100.0/24)
- **Traefik**: 192.168.100.2 (reverse proxy with SSL)

### Frigate Network Options

**Option A: Join traefik-proxy (recommended for remote access)**
```yaml
networks:
  traefik-proxy:
    external: true

services:
  frigate:
    networks:
      traefik-proxy:
        ipv4_address: 192.168.100.20  # Pick unused IP
```

**Option B: Standalone with port mapping (simpler)**
```yaml
services:
  frigate:
    ports:
      - "5000:5000"   # Web UI
      - "8554:8554"   # RTSP restream
      - "8555:8555"   # WebRTC
```

## Hardware Limitations

### No iGPU
The DXP4800+ has no integrated GPU. Frigate detection options:
1. **CPU-only**: Works but higher CPU usage
2. **Coral TPU USB**: Recommended for better performance (~$60)

### Checking USB Devices (for Coral)
```bash
lsusb
# Look for "Google Inc." or "Global Unichip Corp."
```

## Port Conflicts

nginx (UGOS web UI) uses ports 80/443 by default. Already moved to 8080/8443 for Traefik.

If Frigate needs port 80/443 directly (unlikely), coordinate with existing Traefik setup.

## Troubleshooting

### Docker permission denied
```bash
# Check group membership
groups
# Should include: docker

# If not, add user to docker group (needs sudo):
sudo usermod -aG docker your-user
# Then reconnect SSH
```

### Container won't start
```bash
# Check logs
docker logs frigate

# Check if port is in use
sudo netstat -tlnp | grep 5000
```

### Camera stream not connecting
```bash
# Test RTSP (used by this setup - RLN8-410 NVR)
ffprobe "rtsp://admin:YOUR_PASS@nvr-ip:554/h264Preview_01_main"

# Test HTTP-FLV (may not work on all NVR models)
ffprobe "http://nvr-ip/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=YOUR_PASS"
```

**Note**: RLN8-410 doesn't support HTTP-FLV pull streams. Use RTSP.

See [Reolink Camera Guide](REOLINK-CAMERAS.md) for stream type recommendations.
