# Getting Started

## Quick Install

BroadcastStream relies heavily on microservices (Docker Compose) connected via an internal network. You need a modern version of Docker.

```bash
git clone https://github.com/xettri/broadcaststream.git
cd broadcaststream
docker compose up --build -d
```

Ensure all containers report `Started` or `Healthy`:

- `broadcaststream-mediamtx`: Streaming proxy, WebRTC server
- `broadcaststream-nginx`: HLS delivery cache
- `broadcaststream-api`: Node.js webhook engine

### Directory Structure

```
broadcaststream/
├── docker-compose.yml
├── /nginx
│    └── nginx.conf         # Core segment serving tuning
├── /src
│    └── /routes/webhooks   # State management
├── transcode.sh            # FFmpeg ABR Script
├── on_publish.sh           # Webhook Trigger & FFmpeg Process
├── on_unpublish.sh         # Cleanup Memory & Stale Segments
└── /public                 # Broadcaster/Viewer Client
```
