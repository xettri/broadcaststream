# Architecture Overview

BroadcastStream is an event-driven microservices architecture built explicitly to deliver low-latency (glass-to-glass ~2-4s) Adaptive Bitrate (ABR) live video.

It orchestrates three decoupled core services via Docker:

### 1. MediaMTX

A high-performance routing proxy and ingester for WebRTC and RTMP. MediaMTX does not transcode video; instead, it relies on its native webhook hooks (`runOnReady`, `runOnNotReady`) to spin off sub-processes on demand.

### 2. Node.js Tracking API (Port `4000`)

We stripped out bloated state mapping (e.g. Postgres or Redis keys).
The API simply maintains a fast, constantly synchronized array (`Map<string, StreamInfo>`) populated by `/webhook/on-publish` HTTP calls from MediaMTX.

### 3. NGINX Reverse Proxy (Port `8080`)

NGINX provides two critical functions: HLS Segment Delivery and Edge Tunneling.
Cloudflare caching can be unpredictable, breaking the stream. By putting NGINX in front of the local `var/www/hls` volume, we assert aggressive headers (`Cache-Control: immutable`). Because NGINX runs the static UI and API `/api/` subfolders, viewers and tunnels don't need multiple ports open.
