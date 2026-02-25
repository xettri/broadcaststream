# FFmpeg Transcoding Deep Dive

This document breaks down the `transcode.sh` file block by block. This script runs inside the MediaMTX Docker container the moment you hit "Start Streaming" in OBS or the browser.

## The Goal

In live video streaming, clients are on hundreds of different networks: 5G, public Wi-Fi, spotty 3G. Providing only a massive, single 1080p high bitrate broadcast is a bad experience because if their internet slows down even for 4 seconds, the video will freeze ("buffering").

We fix this by using **Adaptive Bitrate Streaming (ABR)**. When the server gets the original 1080p source video, the CPU uses FFmpeg to calculate and generate 4 simultaneous, smaller copies (Renditions) of the video in real-time. Wait, isn't that expensive? Yes, but we use optimized hardware presets that favor Speed over extreme data compression.

### Step 1: Receiving the Stream Key

When MediaMTX calls `transcode.sh`, it passes the stream's name as an argument.

```sh
RTSP_PATH="${1:?Error: RTSP path required}"
STREAM_KEY="${RTSP_PATH#live/}"
```

- **`${1}`**: bash shorthand for "Argument #1". The `?Error:` makes the script crash immediately with an error if no argument was passed.
- **`${#live/}`**: By default, OBS sends RTMP keys with a prefix like `rtmp://.../live/mysecretkey`. This deletes the `live/` part so we only have `mysecretkey` as our folder identifier.

### Step 2: The Safety Wipe

```sh
HLS_ROOT="/var/www/hls"
RTSP_INPUT="rtsp://localhost:8554/${RTSP_PATH}?transport=tcp"
```

- **`OUT_DIR`**: This maps to `/var/www/hls/mysecretkey` where files are saved.
- **`?transport=tcp`**: Critical fix. UDP packets traverse poorly inside docker networks under heavy CPU load (you will see tearing or skipped frames). TCP forces the stream to verify delivery.

```sh
rm -rf "${OUT_DIR}"
```

- **`rm -rf`**: If you stream twice using the same key, we wipe the hard drive cache completely clean the millisecond you hit "Stream" to guarantee NGINX and Cloudflare don't serve your dead video clips from an hour ago.

### Step 3: Manifest Generation

```sh
#Write ABR master playlist explicitly (HLS variant manifest)
cat > "${OUT_DIR}/master.m3u8" << 'EOF'
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=4692000,RESOLUTION=1920x1080,CODECS="avc1.640028..."
1080p/index.m3u8
...
EOF
```

- **`master.m3u8`**: This is the file the video player actually reads. It acts as an index menu that says: _"If your internet speed is faster than 4 Mbps, read from the `1080p` folder. If your speed drops to 1 Mbps, switch dynamically to reading the `480p` folder."_

### Step 4: The Core FFmpeg Process

```sh
ffmpeg \
  -hide_banner -loglevel warning \
  -fflags nobuffer -flags low_delay \
```

- **`nobuffer` & `low_delay`**: Forces FFmpeg to process the video immediately without waiting to build up a queue of frames in memory. By sacrificing 1% of encode efficiency, you save 2+ seconds of latency.

```sh
  -filter_complex \
    "[v:0]split=4[v1][v2][v3][v4]; \
     [v1]scale=1920:1080:flags=lanczos[v1out]; \
     [v2]scale=1280:720... \
```

- **`filter_complex`**: This is where the magic happens. We take the source video `[v:0]` and split it into 4 invisible channels `[v1]-[v4]`.
- **`scale`**: Each split channel is resized using `lanczos`, a high-quality resampling algorithm that produces sharp scaled images, yielding our 4 output variants `[v1out] - [v4out]`.

### Step 5: Encoding Details (1 Rendition example)

Next, we map each unique output into its final form using `libx264` (a fast H.264 CPU encoder supported by 100% of devices).

```sh
  -map "[v2out]" -map "a:0" \
    -c:v:1 libx264 -preset ultrafast -tune zerolatency \
```

- **`[v2out]`**: Grabs the 720p scaled video.
- **`a:0`**: Grabs the single master audio track the broadcaster sent.
- **`preset ultrafast` & `tune zerolatency`**: Instructs the encoder to use the fastest possible algorithms and to ignore B-frames (bi-directional prediction). This is the foundation of Real-Time communication latency.

```sh
    -r 30 -g 30 -sc_threshold 0 \
```

- **`-r 30`**: Frame rate target (fps).
- **`-g 30`**: Group of Pictures (Keyframe Interval). This forces a complete "I-Frame" every 30 frames (1 second). Players can only seek or change video quality on an I-Frame boundary, so setting this low enables ultra-fast playlist switching and live syncing.
- **`-sc_threshold 0`**: Prevents FFmpeg from inserting extra sudden keyframes on scene-changes because sudden keyframe injection ruins HLS segment timings dynamically dividing clips oddly (e.g., 1.2s instead of exactly 1.0s).

```sh
    -b:v:1 0 -crf 23 -maxrate:v:1 2500k -bufsize:v:1 5000k \
```

- **`crf 23`**: Constant Rate Factor. Instead of forcing a static file size, this tells the encoder "maintain a visual quality score of 23" where 0 is lossless and 51 is pixelated garbage, letting the file sizes grow or shrink organically.
- **`maxrate 2500k / bufsize 5000k`**: Even organically, during a scene with millions of colors or confetti falling, video data balloons. We cap this to 2.5 Mbps using `maxrate`. `bufsize` is set to double `maxrate` (5 Mbps limit) which allows milliseconds of spikes without completely devastating quality.

```sh
    -f hls -hls_time 1 -hls_list_size 10 \
    -hls_flags "delete_segments+independent_segments" \
    -hls_segment_filename "${OUT_DIR}/720p/%04d.ts" \
    "${OUT_DIR}/720p/index.m3u8" \
```

- **`-hls_time 1`**: Instructs the encoder to slice the video into exactly 1-second physical files (`.ts` segments) instead of large 6-second defaults.
- **`hls_list_size 10`**: Keep only the last 10 seconds of chunks in the server's `.m3u8` playlist text file.
- **`delete_segments`**: Tells FFmpeg to physically delete the `.ts` files from the server's hard drive as they fall off the playlist (older than 10 seconds). This keeps the hard disk from filling up indefinitely over an 8-hour broadcast.
- **`independent_segments`**: Verifies that every segment starts with its own Keyframe (referencing `-g 30`), heavily ensuring low-latency browsers can sync fast.
- **`%04d.ts`**: Pattern name for segments. The first file is `0000.ts`, the second is `0001.ts`, etc...
