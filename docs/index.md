---
layout: home

hero:
  name: "BroadcastStream"
  text: "Low-Latency HLS & WebRTC Server"
  tagline: "A production-grade microservices stack for sub-3s latency livestreaming over custom custom Cloudflare tunnels."
  actions:
    - theme: brand
      text: Get Started
      link: /guide/getting-started
    - theme: alt
      text: Read Technical Architecture
      link: /architecture/overview

features:
  - title: WebRTC & RTMP Ingest
    details: Supports direct WebRTC ingest via WHIP from modern browsers and RTMP ingest from professional software like OBS and vMix.
  - title: ABR HLS Transcoding
    details: Zero-latency tuned FFmpeg transcoding creates 1080p, 720p, 480p, and 360p adaptive bitrates optimized for all network conditions.
  - title: Real-Time State Management
    details: In-memory Node.js API uses Event Webhooks from MediaMTX. Zero database polling for tracking live stream connection and disconnection events.
  - title: Perfect Edge Cache
    details: Aggressive NGINX tuning removes redundant caching headers and sets strict HLS fragment immutability allowing proxy tunnels to deliver streams smoothly.
---
