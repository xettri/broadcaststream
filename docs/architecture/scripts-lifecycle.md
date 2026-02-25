# Stream Lifecycle Scripts Deep Dive

These two Bash scripts (`on_publish.sh` and `on_unpublish.sh`) are placed inside the `/transcode.sh` image layer. They are automatically triggered by MediaMTX via hooks `runOnReady` and `runOnNotReady` respectively.

This removes the need for infinite database polling `setInterval` loops because Node.js only ever reacts exactly when physical streams start or stop.

---

## 1. `on_publish.sh` (The Starting Hook)

This script is called by MediaMTX when a user connects and successfully sends their first frame of video.

### Receiving MediaMTX Variables

```bash
RAW_PATH="${MTX_PATH:-}"
REMOTE_ADDR="${MTX_REMOTEADDR:-unknown}"
PROTOCOL="${MTX_PROTOCOL:-rtmp}"
STREAM_KEY="${RAW_PATH#live/}"
```

MediaMTX automatically injects these internal environment variables into any script it runs.

- `MTX_PATH`: The full URL path requested (e.g. `live/test`).
- `STREAM_KEY`: We strip `live/` so our API registers the clean "test" name. We use this name to tell FFmpeg where to save the chunks (`.../hls/test/`).

### Launching the Transcoder

```bash
# Launch FFmpeg ABR in the background
/transcode.sh "${RAW_PATH}" "${STREAM_KEY}" &
TRANSCODER_PID=$!
```

MediaMTX is a blocking server. It won't let the user finish "connecting" until this specific script exits.
Therefore, we run the massive `transcode.sh` (FFmpeg process) in the **background** using `&`. We capture its process ID (`$!`) so we can optionally kill it or wait for it later.

### Waiting for HLS to Initialize

```bash
# Wait for master.m3u8 to appear (approx 1-3 seconds)
echo "[on_publish] Waiting for master.m3u8..."
MAX_WAIT=10
t=0
while [ $t -lt $MAX_WAIT ]; do
  if [ -f "/var/www/hls/${STREAM_KEY}/master.m3u8" ]; then
    break
  fi
  sleep 1
  t=$((t+1))
done
```

This is the **"Race Condition Fix."** If we told the API "we are live!" immediately, a viewer might click the video player 0.5s later. But FFmpeg takes about 1-2 seconds to analyze the first few frames of video and actually generate the `master.m3u8` playlist file. The video player would 404 crash because the files don't exist yet! We manually `sleep` and check the hard drive up to 10 times (10s max limit) until the playlist file physically exists.

### Notifying the Global API State

```bash
# Notify API webhook (use wget)
wget -qO- \
  --post-data="name=${STREAM_KEY}&remoteAddr=${REMOTE_ADDR}&proto=${PROTOCOL}" \
  "http://api:4000/webhook/on-publish"
```

Since the `master.m3u8` is legally on the storage volume now, we execute an invisible HTTP POST request using `wget` directly to our internal Node.js API container (which is mapped as `http://api:4000/`). The API memory state is updated, making `test` visible globally on `viewer.html` JSON arrays.

### Awaiting Disconnect

```bash
# Wait for transcoder to finish
wait $TRANSCODER_PID
```

We freeze this publishing script until `transcode.sh` (FFmpeg) completely crashes or finishes transcoding. When a broadcaster stops their stream, MediaMTX forcibly kills the script tree anyway.

---

## 2. `on_unpublish.sh` (The Disconnect Hook)

When a broadcaster clicks "Stop Streaming", or their Internet completely cuts out, MediaMTX triggers `runOnNotReady`, mapping to this script.

```bash
# Notify API webhook immediately
wget -qO- \
  --post-data="name=${STREAM_KEY}&remoteAddr=${REMOTE_ADDR}" \
  "http://api:4000/webhook/on-unpublish"
```

The exact millisecond the connection drops, we ping `http://api:4000/webhook/on-unpublish`. The API immediately runs `activeStreams.delete(name)` so no new visitors click "Watch" on a dead stream.

### The 30s Grace Period Cleanup Logic

```bash
# Delayed cleanup: Wait 30s so viewers can finish the buffered part of the stream.
(
  echo "[on_unpublish] Cleanup scheduled in 30s for ${STREAM_KEY}..."
  sleep 30
  OUT_DIR="/var/www/hls/${STREAM_KEY}"
  if [ -d "${OUT_DIR}" ]; then
    echo "[on_unpublish] Cleanup: Removing expired HLS files for ${STREAM_KEY}"
    rm -rf "${OUT_DIR}"
  fi
) &
```

- **Why the 30s delay?**: Because of HLS chunking latency, if you cut the live stream at exactly 12:00:00, the last 4 seconds of video are still traveling to the viewer's web browser. If we ran `rm -rf` instantly, the viewer's video would hard crash 4 seconds from the end, missing your final "goodbye."
- **Why the `()` and `&`?**: We wrap the sleep/delete code inside `( ... ) &`. This executes the massive delay inside an invisible background bash subshell. This allows `on_unpublish.sh` to instantly exit successfully in 0.1 seconds, unblocking the server so it's ready to handle the broadcaster if they instantly want to reconnect.
- **The `rm -rf`**: After 30s, the background task violently deletes all HLS chunks (`.ts`) and `.m3u8` index playlists. This guarantees neither Cloudflare nor NGINX will serve old ghost data if the broadcaster reconnects with the exact same name 5 minutes later.
