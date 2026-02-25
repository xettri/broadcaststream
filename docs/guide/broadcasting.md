# Broadcasting & Viewing

The system expects inbound video on port `1935` (RTMP) or `8889` (WebRTC). The streams then become available on port `8080/hls...`.

## Broadcaster UI (WebRTC)

Visit your local endpoint `http://localhost:8080/broadcaster.html`.

This UI uses **WHIP** (WebRTC HTTP Ingestion Protocol). WebRTC is low-latency by default, and MediaMTX handles the internal signaling without requiring complex STUN handshakes or local TURN relays on simple edge configurations.

- **Server URL**: `http://localhost:8080/ingest/` (Tunneling via `nginx`)
- **Key**: `webcam`

## Professional Playouts (OBS)

If you require overlays, multiple audio scenes, or a specific broadcast device:

- **Server URL**: `rtmp://localhost:1935/live`
- **Stream Key**: `test`

> We pass the stream to the NGINX stack. HLS requires that any `live/` URL prefix normally seen in OBS be stripped so our API registers the correct ID.

### Viewer UI

The viewer at `viewer.html` leverages Video.js 8 with `videojs/http-streaming` natively built in.
It periodically hits `http://localhost:4000/streams` to update the user with a list of currently active sessions. Clicking a session instantly reloads the `<video>` tag with the appropriate HLS Master `.m3u8`.
