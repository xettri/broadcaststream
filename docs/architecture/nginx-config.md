# NGINX Configuration Deep Dive

This document breaks down the `nginx.conf` file deployed in the `broadcaststream-nginx` Docker container. NGINX plays two vital roles in our stack: it acts as a web server to deliver the video segments (HLS) to users and as a reverse proxy to route internet traffic to our internal Node.js API and MediaMTX server.

## Full Code Context

Here are the most critical parts of the configuration and exactly what each line does.

### Global & HTTP Block

```nginx
worker_processes auto;
events { worker_connections 1024; }
```

- **`worker_processes auto`**: Tells NGINX to create one worker process per CPU core. If your server has 4 cores, NGINX uses 4 workers to handle traffic efficiently.
- **`worker_connections 1024`**: Each of those workers can handle 1,024 simultaneous connections (e.g. 4000+ total simultaneous viewers).

```nginx
sendfile on;
tcp_nopush on;
tcp_nodelay on;
proxy_buffering off;
```

- **`sendfile on`**: Directly copies files from the hard drive (SSD) to the network card, bypassing the CPU. This is the secret to high-performance video delivery.
- **`tcp_nopush` / `tcp_nodelay`**: Optimizes how TCP packets are sent over the network to reduce latency for the end user.
- **`proxy_buffering off`**: **CRITICAL**. This tells NGINX _not_ to hold onto data before sending it. In a low-latency live stream, we want video chunks pushed to the user the millisecond they are generated.

### HLS Video Delivery (Port 8080)

The server block listens on port 8080. When a user requests an HLS file (`.m3u8` playlist or `.ts` video chunk), this block handles it.

```nginx
location /hls {
    add_header 'Access-Control-Allow-Origin'  '*' always;
```

- **Wide open CORS**: `*` means ANY website can embed the video player and fetch chunks from this server without the browser blocking it.

```nginx
location ~* \.m3u8$ {
    add_header 'Cache-Control' 'no-cache, no-store, must-revalidate' always;
}
```

- **The Playlist Rule**: `.m3u8` files are text files that constantly update as new video is recorded. We force the browser (`no-cache`) to NEVER store this file. If the browser cached this file, the stream would immediately freeze because the browser would never fetch the newest video segments.

```nginx
location ~* \.ts$ {
    add_header 'Content-Type' 'video/mp2t' always;
    add_header 'Cache-Control' 'public, max-age=3600, immutable' always;
}
```

- **The Video Segment Rule**: `.ts` files are physical chunks of recorded video (e.g., seconds 0 to 1 of your stream). Once a piece of video happens, it never changes.
- **`immutable`**: We tell the browser it is safe to cache this chunk for 1 hour (`3600`). This saves your server massive amounts of bandwidth if a user rewinds the live stream.

### Cloudflare Tunnel Proxies

Because we only expose one URL via Cloudflare (e.g., `https://your-tunnel.trycloudflare.com`), NGINX must catch specific URLs and secretly forward them internally to other Docker containers.

```nginx
location /api/ {
    proxy_pass http://api:4000/;
}
```

- **Node API**: If the frontend Javascript needs the list of active streams, it calls `/api/streams`. NGINX silently forwards this request to the `api` container on port `4000`, grabs the JSON response, and sends it back to the user.

```nginx
location /ingest/ {
    proxy_pass http://mediamtx:8889/;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

- **WebRTC WHIP Ingest**: When a broadcaster uses their webcam in the browser, WebRTC signaling hits `/ingest/`.
- **`Upgrade`**: WebRTC requires WebSockets (persistent open connections) for signaling. The `Upgrade` headers tell NGINX to upgrade the standard HTTP request into a persistent real-time connection and route it to MediaMTX on port 8889.
