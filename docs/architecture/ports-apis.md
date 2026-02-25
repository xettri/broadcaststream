# Ports, Services & API Details

This document covers what ports the system uses, what each Docker container does, and a full reference of the internal API.

### 1. Docker Services & Internal Ports

The stack has 3 main services located within the Docker network `shopstream_default`.

- **`broadcaststream-mediamtx`** (The Ingester):
  - **Ingest Protocol**: `RTMP, WHIP, RTSP`
  - **Port 1935 (TCP)**: Public RTMP Port. Used by OBS, vMix, and mobile broadcast software.
  - **Port 8889 (TCP)**: WebRTC/WHIP ingest port. Proxied by NGINX internally.
  - **Port 8554 (TCP)**: Internal RTSP protocol port. Used internally by `transcode.sh` to grab the raw video via `rtsp://localhost:8554/...`.

- **`broadcaststream-api`** (The State Manager):
  - **Internal Port 4000 (TCP)**: Runs Express.js. Serves as a unified state database using an in-memory `Map` that tracks exactly who is broadcasting. This port is completely blocked to the outside internet.

- **`broadcaststream-nginx`** (The Cache & Tunnel):
  - **Public Port 8080 (TCP)**: The only port necessary for Cloudflare tunnels. Any request to standard HTTP endpoints flows through this, including static `.html` files in `/public`, `.ts` video chunks in `/var/www/hls`, and reverse-proxying API endpoints.

---

## 2. API Endpoints Reference

The Node.js Express API (`broadcaststream-api`) acts as the state manager for the entire application.

### `GET /health`

- **Visibility**: Public via Cloudflare (`http://.../api/health`) or NGINX (`:8080/api/health`).
- **Purpose**: Orchestrator readiness. Used internally by Docker Compose's `HEALTHCHECK` to ensure the Node instance is ready to receive requests.
- **Return**: JSON `{"status": "ok", "service": "broadcaststream-api"}`

### `GET /streams`

- **Visibility**: Public via Cloudflare / NGINX reverse proxy (`/api/streams`).
- **Purpose**: Used by the Viewers (`viewer.html`). Returns a JSON array of every user currently broadcasting. NGINX enables wide-open CORS so any frontend app on the internet can call this endpoint.
- **Return Structure**:
  ```json
  {
    "success": true,
    "data": {
      "count": 1,
      "streams": [
        {
          "streamKey": "test",
          "startedAt": "2026-02-26T...",
          "clientIp": "192.168.1.5",
          "masterPlaylistUrl": "http://localhost:8080/hls/test/master.m3u8",
          "qualities": [ ... ]
        }
      ]
    }
  }
  ```

### `POST /webhook/on-publish`

- **Visibility**: Internal completely. NGINX refuses to proxy this. Only `broadcaststream-mediamtx` can call this.
- **Payload**: `application/x-www-form-urlencoded`
  - `name`: The stream key (e.g. "test").
  - `remoteAddr`: Broadcaster IP address.
- **Purpose**: When `on_publish.sh` successfully begins creating video files, it POSTs this webhook. The API adds the stream to its memory `Map` making the stream globally visible.

### `POST /webhook/on-unpublish`

- **Visibility**: Internal completely.
- **Payload**: `application/x-www-form-urlencoded` (`name`, `remoteAddr`).
- **Purpose**: When the stream disconnects, `on_unpublish.sh` triggers this. The API instantly removes the session from the `Map`, causing `viewer.html` pages to stop seeing the "test" stream.
