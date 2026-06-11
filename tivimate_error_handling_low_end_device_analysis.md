# TiViMate Low-End Device Optimizations, Error Handling & External Players — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s memory and decoder optimization strategies for low-spec TV boxes, fallback error taxonomies, and external media player intent integrations**. It outlines how the app adapts to restricted TV chipsets, recovers from stream crashes, and transfers playback states to third-party player packages.

---

## 1. LOW-END DEVICE OPTIMIZATIONS (TV PROFILE SYSTEM)

Low-cost Android TV sticks (e.g., standard Firesticks, budget Chromecast dongles) have extremely limited memory (1.5GB RAM) and weak CPU chipsets. Unoptimized apps will suffer from continuous Out-of-Memory (OOM) crashes and rendering lag.

TiViMate adapts dynamically to hardware limitations by querying system capabilities at startup.

### The Memory Class Profiler
The application queries the device's memory class via `ActivityManager.getMemoryClass()` and scales its caching systems:

```
[ TV App Cold Start ] ──► [ Query activityManager.getMemoryClass() ]
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼ (High Memory >= 256MB)                                    ▼ (Low Memory < 128MB)
  - EPG RAM TTL: 90 Minutes.                                  - EPG RAM TTL: 20 Minutes (Flush quickly).
  - Glide Logo Cache: 250MB.                                  - Glide Logo Cache: 50MB (Reclaim aggressive).
  - Multiview Screen Limit: 9 Screens.                         - Multiview Screen Limit: 2 Screens.
  - Slide & Zoom animations: ENABLED.                         - Slide & Zoom animations: DISABLED.
```

### Decoder Resource Fallbacks
Hardware video decoders (MediaCodec) on cheap TV chips can only decode one HD H.264 stream at a time.
*   **The Trap**: Initializing multiple streams in Multiview or zapping rapidly will throw `ExoPlaybackException` with code `DECODER_INIT_FAILED`.
*   **The Fallback Secret**:
    Upon catching a decoder initialization failure, TiViMate's player engine automatically triggers a **Software Decoder Fallback** (e.g., utilizing the `ffmpeg-extension` or software codecs embedded in ExoPlayer). While software decoding consumes more CPU, it prevents the app from crashing and keeps the stream active.

---

## 2. ERROR HANDLING TAXONOMY & AUTO-RETRY FLOWS

A TV application must present clear, action-oriented error messages instead of technical stack traces that confuse living-room viewers.

### The TiViMate Error Hierarchy
1.  **Playlist Parse Errors**:
    *   *Cause*: Broken URLs or missing headers.
    *   *Message*: `"Failed to update playlist. Please check your URL or connection."`
2.  **Streaming Playback Errors (403, 404, 503)**:
    *   *Cause*: Token expiry, server load, or invalid URLs.
    *   *Auto-Retry System*: The engine attempts a **3-step retry with exponential backoff** (delaying 1s, 2s, and 4s). If all retries fail, it displays: `"An error occurred: Code 403. Retrying..."`
3.  **Authentication Drops**:
    *   *Cause*: Expired Stalker MAC sessions.
    *   *Silent Recovery*: Catch the 403 error, execute the background Stalker handshake in `playlists.stalker_token`, re-fetch the stream link, and reload.

---

## 3. EXTERNAL PLAYER INTENT INTEGRATION

Some streams use complex proprietary video formats that require external players (VLC, MX Player) to render correctly.

### The Intent Dispatcher
When a user clicks "Play in External Player," TiViMate halts its internal ExoPlayer engine and dispatches a standard Android **Implicit Intent** holding the target video stream URI:

```kotlin
// Replicated safe External Player Intent Dispatcher
fun playInExternalPlayer(context: Context, streamUrl: String, userAgent: String?) {
    val intent = Intent(Intent.ACTION_VIEW).apply {
        setDataAndType(Uri.parse(streamUrl), "video/*")
        putExtra("title", "Live Broadcast")
        // Inject User-Agent headers (aggressive provider requirements)
        userAgent?.let {
            val headers = Bundle().apply { putString("User-Agent", it) }
            putExtra("android.media.intent.extra.HTTP_HEADERS", headers)
        }
        flags = Intent.FLAG_ACTIVITY_NEW_TASK
    }
    
    try {
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        Toast.makeText(context, "No compatible external player found.", Toast.LENGTH_SHORT).show()
    }
}
```

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Map your video play intents to inject standard User-Agent headers inside the HTTP bundle to prevent IPTV provider authentication rejections.
*   Present user-friendly error strings instead of raw stack traces.

### Medium-Risk Actions
*   Implement dynamic cache scaling based on the device's memory class (`ActivityManager.getMemoryClass()`), lowering Glide image limits on 1.5GB RAM TV boxes.
*   Implement software decoder fallbacks (FFmpeg audio/video extensions) inside your ExoPlayer initialization routines.

### High-Risk Actions
*   Launching multi-player views concurrently on low-spec hardware without checking core counts or memory classes, which triggers immediate device freezing.

### What to Measure First
*   Monitor system heap usage. Low-memory devices must maintain a constant **under-80MB heap usage** to prevent garbage collection spikes.

### What Not to Copy Directly
*   Do not copy custom intent handlers or internal player overrides.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Dynamic memory-aware cache scaling and software decoder fallback paths to support low-end TV hardware.
