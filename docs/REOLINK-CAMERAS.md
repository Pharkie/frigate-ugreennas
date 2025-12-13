# Reolink Camera Configuration for Frigate

Reolink cameras have inconsistent features across models. This guide covers stream types, settings, and URL formats for optimal Frigate integration.

## Prefer Home Assistant API

If Reolink is already integrated in Home Assistant, **use the HA API** to query camera info rather than the Reolink API directly. The HA integration is more reliable and provides a cleaner interface.

```bash
# List all cameras via HA API
source .env
curl -s -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/states" | \
  python3 -c "import sys,json; [print(s['entity_id'], s['attributes'].get('friendly_name','')) for s in json.load(sys.stdin) if s['entity_id'].startswith('camera.')]"
```

## Stream Type Selection

| Camera Resolution | Camera Generation | Recommended Stream | Notes |
|------------------|-------------------|-------------------|-------|
| 5MP or lower | All | `http-flv` | Stream is H264 |
| 6MP or higher | Latest (Duo3, CX-8##) | `http-flv` with ffmpeg 8.0, or `rtsp` | Uses http-flv-enhanced over H265, requires ffmpeg 8.0 |
| 6MP or higher | Older (RLC-8##) | `rtsp` | |

**Key takeaway**: HTTP video streams are generally more reliable for Reolink cameras than RTSP.

## Recommended Camera Settings

Configure these in the Reolink app/web interface for best Frigate compatibility:

1. **Bitrate Control**: Set to **"On, fluency first"**
   - This enables CBR (constant bit rate)
   - More predictable stream for Frigate to process

2. **Interframe Space**: Set to **1x**
   - Sets iframe interval equal to frame rate
   - Improves seek performance and reduces detection latency

## Stream URL Formats

### Direct Camera Connection

```yaml
# HTTP-FLV (recommended for 5MP or lower)
http://camera-ip/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=YOUR_PASS

# HTTP-FLV substream (for detect)
http://camera-ip/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=admin&password=YOUR_PASS

# RTSP main stream
rtsp://admin:YOUR_PASS@camera-ip:554/h264Preview_01_main

# RTSP substream
rtsp://admin:YOUR_PASS@camera-ip:554/h264Preview_01_sub
```

### Cameras Connected via Reolink NVR

Use `channel[0..15]` in the URL for NVR-connected cameras:

```yaml
# Channel 0 (first camera)
http://nvr-ip/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=YOUR_PASS

# Channel 1 (second camera)
http://nvr-ip/flv?port=1935&app=bcs&stream=channel1_main.bcs&user=admin&password=YOUR_PASS

# RTSP via NVR (less reliable on some hardware versions)
rtsp://admin:YOUR_PASS@nvr-ip:554/h264Preview_01_main
```

**Note**: HTTP streams through NVR are confirmed working on older hardware (RLN16-410) with multiple camera types. RTSP via NVR is not always reliable.

## Example Frigate Configuration

```yaml
cameras:
  # Direct connection - 5MP camera
  front_door:
    ffmpeg:
      inputs:
        - path: http://192.168.0.50/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=admin&password=YOUR_PASS
          roles:
            - detect
        - path: http://192.168.0.50/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=YOUR_PASS
          roles:
            - record

  # Via Reolink NVR - channel 2
  backyard:
    ffmpeg:
      inputs:
        - path: http://192.168.0.10/flv?port=1935&app=bcs&stream=channel2_ext.bcs&user=admin&password=YOUR_PASS
          roles:
            - detect
        - path: http://192.168.0.10/flv?port=1935&app=bcs&stream=channel2_main.bcs&user=admin&password=YOUR_PASS
          roles:
            - record
```

## Dual-Stream Setup

For best performance, use two streams per camera:

| Stream | Resolution | Use in Frigate | Why |
|--------|-----------|----------------|-----|
| Main (`channel0_main`) | Full resolution | `record` | High quality recordings |
| Substream (`channel0_ext`) | Lower resolution | `detect` | Faster AI processing |

## Troubleshooting

### Test Stream Connectivity

```bash
# Test HTTP-FLV stream
ffprobe "http://camera-ip/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=YOUR_PASS"

# Test RTSP stream
ffprobe "rtsp://admin:YOUR_PASS@camera-ip:554/h264Preview_01_main"
```

### Common Issues

1. **Stream drops frequently**
   - Try HTTP-FLV instead of RTSP
   - Enable CBR on camera settings

2. **High latency**
   - Set iframe interval to 1x
   - Use substream for detection

3. **ffmpeg errors with H265**
   - Newer 6MP+ cameras may output H265
   - Requires ffmpeg 8.0 for http-flv-enhanced
   - Alternative: Force H264 in camera settings or use RTSP

4. **NVR streams unreliable**
   - HTTP streams are more reliable than RTSP through NVR
   - If RTSP needed, test on your specific NVR hardware version

5. **HTTP-FLV not working on NVR**
   - Some NVR models (e.g., RLN8-410) don't support HTTP-FLV pull streams
   - RTMP port may be for outbound push only, not inbound FLV requests
   - Check if RTMP is enabled: NVR → Settings → Network → Advanced → Server Settings
   - If HTTP-FLV fails even with RTMP enabled, use RTSP instead

   ![Reolink RTMP Settings](images/Reolink_RTMP.png)

## References

- [Frigate Reolink Documentation](https://docs.frigate.video/configuration/camera_specific#reolink-cameras)
- [Reolink HTTP-FLV Discussion](https://github.com/blakeblackshear/frigate/discussions/)
