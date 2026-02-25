# MediaMTX Configuration Deep Dive

This document breaks down the `mediamtx.yml` configuration file. MediaMTX is the brain of our live streaming infrastructure—it receives video feeds, acts as a proxy, and triggers scripts that run FFmpeg or notify our Node.js API.

## Core Settings

```yaml
###############################################
# Global settings

# Log Level and Destinations
logLevel: info
logDestinations: [stdout]
```

These basic settings ensure that any errors, warnings, or incoming connections are printed directly to the Docker console (`docker logs broadcaststream-mediamtx`) rather than being buried in a file.

### API & Server Metrics

```yaml
###############################################
# API / metrics

api: yes
apiAddress: :9997
```

MediaMTX comes with a built-in control plane API. We enable it here (`api: yes`) on port 9997. While we rely heavily on Webhooks for stream state management, having the direct API open allows for manual intervention, forced kick commands, or scraping advanced metrics into Prometheus if needed in the future.

### RTMP Server Settings (OBS)

```yaml
###############################################
# RTMP server

rtmp: yes
rtmpAddress: :1935
```

This block enables the RTMP (Real-Time Messaging Protocol) server. `1935` is the global standard port for RTMP. When the broadcaster types `rtmp://localhost:1935/live/test` into OBS, this is the module that accepts the incoming video layout.

### WebRTC Server Settings (Browser Ingest)

```yaml
###############################################
# WebRTC server

webrtc: yes
webrtcAddress: :8889
webrtcICEServers: [stun:stun.l.google.com:19302]
```

This block is incredibly important for "Zero-Install" browser broadcasting (like our `broadcaster.html` UI). We run the WHIP (WebRTC HTTP Ingestion Protocol) server on port `8889`.

- **`webrtcICEServers`**: WebRTC is a peer-to-peer protocol that utilizes ICE candidates to figure out how to pass traversing packets through a broadcaster's home NAT/Router. We provide Google's free STUN server as a fallback tool so the broadcaster's browser can compute its public IP and complete the handshake with our server.

> Note: We specifically disabled `.udp` settings in the broader config because Cloudflare HTTP tunnels only proxy TCP traffic.

### The Path Architecture (Where the magic happens)

The `paths:` block is where MediaMTX gets truly powerful. It defines rules for what happens when a stream connects to a specific URL path.

```yaml
###############################################
# Path settings

paths:
  # Catch-all: triggers on ANY incoming stream key
  all_others:
```

By using `all_others:`, we define a wildcard rule. Whether the user streams to `/live/test`, `/live/mywebcam`, or any random string, the following rules will trigger automatically.

```yaml
# Start the encoder and notify the API when stream is published
runOnReady: /on_publish.sh
```

- **`runOnReady`**: This is a native MediaMTX hook. The exact millisecond OBS finishes its RTMP handshake and the first frame of video is fully received in RAM, MediaMTX executes the `on_publish.sh` script. This script tells the Node.js API "We are live!" and starts the FFmpeg transcoding sub-process.

```yaml
# Stop the encoder and notify the API when stream ends
runOnNotReady: /on_unpublish.sh
```

- **`runOnNotReady`**: Similarly, when the broadcaster hits "Stop Streaming" or the internet cuts out for more than ~10 seconds, this hook fires. The `on_unpublish.sh` script tells the Node API to remove the stream from the active database and manages the aggressive hard drive cleanup (so old data isn't cached).

```yaml
# Allows the script to replace standard MediaMTX behavior
runOnReadyRestart: true
```

- **`runOnReadyRestart`**: This tells MediaMTX that the script being run (our FFmpeg transcoder) is going to completely take over reading the stream data. If the script crashes for any reason, this command tells MediaMTX to immediately try restarting the `on_publish.sh` sub-process to recover the video feed.
