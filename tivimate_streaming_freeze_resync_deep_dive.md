# TiViMate Streaming, Freeze Detection, Source Rechecking & Resyncing — Deep Dive

This architectural analysis provides a highly detailed breakdown of how **TiViMate** achieves absolute streaming stability in living-room environments. It explains the exact mechanics behind its low-latency playback buffering, video freeze detection, source rechecking, dynamic authentication refreshes, and exponential backoff retry flows.

---

## 1. THE ARCHITECTURAL PLAYBACK ENGINE (HYBRID CORE)

To understand how TiViMate handles video freezes, disconnections, and resyncs, we must first look at its underlying hybrid framework.

```
       ┌────────────────────────────────────────────────────────┐
       │                 Android JVM UI Layer                   │
       │  (EPG Canvas, Aspect Ratio Views, Leanback Dialogs)   │
       └───────────────────────────┬────────────────────────────┘
                                   │ (JNI Interface Bindings)
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │                Native C++ Core Engine                  │
       │    (libtvplayer.so / libmessageguard.so Wrapper)      │
       │ * State Machine       * Token Security / Handshakes    │
       │ * Socket Watchdogs    * Stream URL Encryption / Decrypt│
       └───────────────────────────┬────────────────────────────┘
                                   │ (Media3 Extension Factory)
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │                 ExoPlayer Playback Layer               │
       │  (Custom LoadControl, OkHttp DataSource, AudioSink)   │
       └────────────────────────────────────────────────────────┘
```

### The JVM-Native Decoupling
Standard Android IPTV apps implement their entire player wrapper, network retry loops, and token parsing in pure Java or Kotlin. In living-room environments, this causes significant problems:
* **Garbage Collection (GC) Stutters**: Frequent allocation of short-lived objects (e.g. during stream packet parsing or network state changes) triggers JVM GC pauses. These pauses cause brief rendering stutters (micro-freezes) that ruin the premium TV experience.
* **Obfuscation and Protection**: Reverse engineering TiViMate reveals that classes in the main package `ar.tvplayer.tv` (such as `LibTvPlayerApplication` and `MessageGuardException`) are highly protected boot wrappers. 
* **Native C++ Watchdog Core**: TiViMate bypasses JVM bottlenecks by offloading its state machine, socket keepalives, API handshakes, and URL decryption algorithms to a custom C++ native library (`libtvplayer.so`). The Java layer simply instantiates ExoPlayer wrappers (customized inside packages like `p392` and `p457` to obfuscate standard Media3 callbacks) and communicates status updates down to the native watchdog core over a high-speed JNI bridge.

---

## 2. STUCK BUFFER & VIDEO FROZEN DETECTION (THE BUFFERING WATCHDOG)

In Live IPTV, a network freeze is distinct from a server disconnect.
* **Server Disconnect**: The server explicitly closes the socket (returning TCP FIN/RST) or the host network completely drops. ExoPlayer immediately catches this as an `IOException` and throws an `ExoPlaybackException` to the player listener.
* **Stream Freeze (Perpetual Buffering)**: The server stops sending packet data, but leaves the TCP socket open (a hung connection). From ExoPlayer's perspective, the buffer runs dry, so it transitions to `STATE_BUFFERING` and sits waiting forever for new packets. No exception is thrown, resulting in a **frozen video frame** and a spinning loading circle.

TiViMate solves this with an aggressive, dual-layer **Buffering Watchdog System**.

```
[ Video Playing: STATE_READY ]
       │
       ▼ (Packets Stop Arriving: Socket Hung)
[ Player enters STATE_BUFFERING ] ──► Starts Watchdog Timer (Default: 5,000ms)
       │
       ├─────────────────────────────────┐
       ▼ (Packets Resume before 5s)      ▼ (Timer Expires at 5,000ms: Frozen!)
[ Cancel Watchdog / Resume ]       [ Trigger Silent Resync ]
                                         │
                                         ▼
                                   * Dispose active socket
                                   * Query Native Core for Fresh Link
                                   * Re-prepare ExoPlayer instance
```

### The Watchdog Mechanics
1. **The State Listener**: The Player Controller listens for playback state changes. If the state transitions to `STATE_BUFFERING`, a scheduled watchdog coroutine or `Handler` callback is registered with a strict timeout (typically **5,000 milliseconds**).
2. **First-Frame Safeguard**: On slower TV sticks, a hardware decoder might lock up during a rapid channel zap. To detect this, the watchdog checks if `onRenderedFirstFrame` is called within 5 seconds of the player transitioning to `STATE_READY`. If the player is in the ready state but no frames are actually rendered to the surface, the watchdog marks the stream as frozen.
3. **The Silent Resync Trigger**: If the watchdog timer expires while the player is still stuck in `STATE_BUFFERING` or has failed to render its first frame, it declares a freeze. Instead of showing an error toast, it initiates a **Silent Resync**:
   * It tears down the active TCP connection.
   * It calls the native backend to re-verify the streaming credentials and request a fresh dynamic play link.
   * It updates the player source and calls `player.prepare()` to force a clean socket handshake.

---

## 3. SOCKET LAYER & CUSTOM LOADCONTROL CONFIGURATION

To make zapping and buffering recovery instantaneous, TiViMate bypasses standard ExoPlayer configurations (which are tuned for mobile YouTube/Netflix buffering and wait for 15-50 seconds of video before starting playback).

TiViMate implements a custom `LoadControl` configuration (`p392.C4655`) and binds it to the **Media3 OkHttp Extension DataSource** (`p338.C4227`).

```
[ ExoPlayer Instance ] ◄─── Binds ─── [ Custom LoadControl (C4655) ]
       │                                       │
       ▼ (OkHttp Socket Pipe)                  ▼ (Aggressive Buffer Bounds)
[ OkHttp DataSource (C4227) ]            * minBufferMs = 1,000ms
       │                                 * maxBufferMs = 2,500ms
       ▼ (Shared Socket Pool)            * bufferForPlayback = 500ms
[ Pre-Warmed TCP Sockets ]               * bufferAfterRebuffer = 1,000ms
```

### The Buffering Bounds (`p392.C4655.java`)
In `C4655.java`, TiViMate's constructor restricts buffers using aggressive, low-latency values:
* **`minBufferMs` (1,000ms)**: Tells ExoPlayer it only needs 1 second of media data in its memory buffer to remain stable.
* **`maxBufferMs` (2,500ms)**: Restricts the maximum buffer to 2.5 seconds. If zapping away, this prevents ExoPlayer from downloading megabytes of unneeded stream data, saving significant memory heap and bandwidth on cheap TV boxes.
* **`bufferForPlaybackMs` (500ms)**: The video decoder starts rendering frames as soon as **half a second** of media data enters the socket queue, achieving near-instantaneous zapping.
* **`bufferForPlaybackAfterRebufferMs` (1,000ms)**: In case of network drops, the player only waits for 1 second of buffer before resuming playback, preventing long buffering wheels.

### OkHttp Socket Pooling (`p338.C4227.java`)
In `C4227.java`, TiViMate implements custom OkHttp datasource pipelines instead of standard `HttpURLConnection` wrappers:
1. **TCP Connection Keep-Alive**: When a user zaps channels on the same provider server, OkHttp re-uses the existing TCP connection pool. It bypasses the TLS/SSL negotiation handshake entirely, saving **200–500ms** of latency per channel change.
2. **Aggressive Connection/Read Timeouts**: Connection and read timeouts are set to **5 seconds** (compared to the standard 15-30s). If a stream URL is dead, the socket fails instantly, allowing the auto-retry engine to kick in immediately rather than leaving the user staring at a black screen.

---

## 4. THE AUTO-RETRY ENGINE & EXPONENTIAL BACKOFF

When a stream formally fails (e.g. throwing a socket timeout, connection refused, or HTTP 403/404/503), TiViMate's player controller intercepts the exception and executes its **3-Step Auto-Retry Engine**.

```
[ Active Stream Playback Failure Caught ]
                 │
                 ▼
[ Retry Attempt #1: 1s Delay ] ──► Success ──► Resume Playback
                 │
                 ▼ (Failure)
[ Retry Attempt #2: 2s Delay ] ──► Success ──► Resume Playback
                 │
                 ▼ (Failure)
[ Retry Attempt #3: 4s Delay ] ──► Success ──► Resume Playback
                 │
                 ▼ (Failure)
[ Halt Engine & Display "An error occurred: Code XXX. Retrying..." ]
```

### The Auto-Retry Mechanics
1. **Friendly Error Interceptor**: Instead of throwing raw stack traces or JVM crash logs to the screen, TiViMate maps all media exceptions (`ExoPlaybackException`, `HttpDataSourceException`, and `InvalidResponseCodeException`) to friendly error codes (e.g., Code 403, Code 404, Code 503).
2. **3-Step Exponential Backoff**: When a stream drop is caught, the player engine attempts to re-initialize the media source up to **3 consecutive times** with a progressive delay:
   * **Attempt 1**: Waits **1 second** before retrying.
   * **Attempt 2**: Waits **2 seconds** before retrying.
   * **Attempt 3**: Waits **4 seconds** before retrying.
3. **The "Retrying..." Overlay**: During these three retry attempts, the stream is kept in a background reload state. If the third attempt fails, the engine displays a premium, non-intrusive UI overlay card: `"An error occurred: Code 403. Retrying..."`. It leaves this loop running in the background at a fixed interval (e.g., every 10 seconds) so that if the provider's server comes back online, the broadcast resumes automatically without requiring manual remote clicks from the user.

---

## 5. SILENT SESSION TOKEN RECOVERY & DYNAMIC LINK RESOLVING

Modern IPTV providers (especially Xtream Codes and Stalker Portals) secure their streams using time-locked tokens or temporary signed URLs.
* **The Problem**: A Stalker Portal requires calling `/portal.php?type=itv&action=create_link&cmd=...` to generate a temporary, signed stream URL. These links are typically only valid for **60 minutes**. Once they expire, the stream drops and returns an `HTTP 403 Forbidden` error.

TiViMate handles this with a seamless **Silent Session Token Recovery Loop**:

```
[ Playback returns HTTP 403 Forbidden ]
                   │
                   ▼
[ Intercept error in Player Controller ]
                   │
                   ▼
[ Dispatch Background Coroutine Task ]
  1. Re-initiate MAC handshake: /portal.php?action=handshake
  2. Parse fresh token and update `playlists.stalker_token` in Room
  3. Re-request temporary signed streaming link: /portal.php?action=create_link
                   │
                   ▼
[ Hot-Swap Streaming Source URL in active ExoPlayer instance ]
                   │
                   ▼
[ Call player.prepare() & resume playback seamlessly ]
```

### The Silent Recovery Flow
1. **The 403 Interceptor**: When ExoPlayer encounters an `InvalidResponseCodeException` with code `403` or `401`, it passes it to the player listener.
2. **Silent Handshake Dispatcher**: The player controller catches this token expiry. Instead of stopping playback, it fires a background async worker to execute a quiet handshake:
   * It posts a handshake command to `/portal.php?type=itv&action=handshake&mac=00:1A:79...`.
   * It extracts the fresh session `token` and transactionally writes it to the SQLite database column `playlists.stalker_token` (`C3220.java` line 172).
   * It uses this new token to request a fresh signed stream link via `action=create_link`.
3. **Seamless Hot-Swap**: Once the new streaming URL is resolved, the player controller hot-swaps the active `MediaSource` URL inside the running ExoPlayer instance. It calls `player.prepare()` to initiate the new socket connection. The user only experiences a brief 1-2 second buffering wheel instead of a full playback crash.

---

## 6. HARDWARE DECODER EXHAUSTION SAFEGUARDS

On budget TV boxes with only 1.5GB of RAM (e.g., standard Firesticks), zapping channels rapidly or launching multiple streams in Multiview will exhaust the device's hardware media decoders (`MediaCodec`).
* **The Crash**: This triggers a critical `ExoPlaybackException` with code `DECODER_INIT_FAILED`. In cheap players, this crashes the app to the home screen.
* **TiViMate's Software Decoder Fallback**:
  Upon catching a decoder initialization failure, TiViMate's engine intercepts the exception and immediately instantiates a **Software Decoder Fallback** (utilizing the `ffmpeg-extension` or software video/audio codecs bundled inside the player bundle). While software decoding consumes more CPU resources, it successfully decodes the stream, resolving the video freeze and keeping the player active.

---

## 7. ACTIONABLE IMPLEMENTATION BLUEPRINTS FOR PUREFUSIONIPTV

To implement these premium stabilization techniques, we can integrate the following Kotlin blueprints.

### Blueprint A: Low-Latency LoadControl Buffering Builder
```kotlin
package com.purefusioniptv.player

import androidx.media3.exoplayer.DefaultLoadControl
import androidx.media3.exoplayer.LoadControl

object PlayerBufferTuning {
    /**
     * Replicates TiViMate's low-latency, aggressive buffer parameters.
     * Restricts allocations to prevent JVM heap exhaustion and zapping lag.
     */
    fun createCustomLoadControl(): LoadControl {
        return DefaultLoadControl.Builder()
            .setBufferDurationsMs(
                1000, // minBufferMs: Start ready with 1s of data
                2500, // maxBufferMs: Max queue depth to save heap RAM
                500,  // bufferForPlaybackMs: Instant zapping zap (half second)
                1000  // bufferForPlaybackAfterRebufferMs: Fast stutter recovery
            )
            .setPrioritizeTimeOverSizeThresholds(true)
            .build()
    }
}
```

### Blueprint B: OkHttp DataSource Factory (Connection Pooling & Socket Timeouts)
```kotlin
package com.purefusioniptv.network

import androidx.media3.datasource.DataSource
import androidx.media3.datasource.okhttp.OkHttpDataSource
import okhttp3.ConnectionPool
import okhttp3.OkHttpClient
import java.util.concurrent.TimeUnit

object PlayerNetworkConfig {
    private val sharedHttpClient: OkHttpClient by lazy {
        OkHttpClient.Builder()
            .connectTimeout(5, TimeUnit.SECONDS) // Aggressive connect timeout (fast fail)
            .readTimeout(5, TimeUnit.SECONDS)    // Fast read timeout to detect hung sockets
            .writeTimeout(5, TimeUnit.SECONDS)
            .connectionPool(
                ConnectionPool(
                    maxIdleConnections = 10,
                    keepAliveDuration = 5,
                    timeUnit = TimeUnit.MINUTES
                )
            ) // Pre-warms sockets for zapping within the same provider domain
            .retryOnConnectionFailure(true)
            .build()
    }

    fun createDataSourceFactory(userAgent: String): DataSource.Factory {
        return OkHttpDataSource.Factory(sharedHttpClient)
            .setUserAgent(userAgent)
    }
}
```

### Blueprint C: Buffering Watchdog (Video Freeze & Stuck Buffer Detector)
```kotlin
package com.purefusioniptv.player

import android.os.Handler
import android.os.Looper
import androidx.media3.common.Player

class PlaybackWatchdog(
    private val player: Player,
    private val freezeTimeoutMs: Long = 5000L,
    private val onFreezeDetected: () -> Unit
) {
    private val handler = Handler(Looper.getMainLooper())
    private var isWatchdogRunning = false

    private val watchdogRunnable = Runnable {
        if (player.playbackState == Player.STATE_BUFFERING) {
            // Watchdog triggered! Player has been stuck buffering for 5 seconds.
            isWatchdogRunning = false
            onFreezeDetected()
        }
    }

    fun startTracking() {
        if (isWatchdogRunning) return
        
        if (player.playbackState == Player.STATE_BUFFERING) {
            isWatchdogRunning = true
            handler.postDelayed(watchdogRunnable, freezeTimeoutMs)
        }
    }

    fun stopTracking() {
        handler.removeCallbacks(watchdogRunnable)
        isWatchdogRunning = false
    }

    /**
     * Binds to the standard Player.Listener to track state transitions.
     */
    fun getPlayerListener(): Player.Listener {
        return object : Player.Listener {
            override fun onPlaybackStateChanged(playbackState: Int) {
                when (playbackState) {
                    Player.STATE_BUFFERING -> {
                        // Start freeze countdown
                        startTracking()
                    }
                    Player.STATE_READY, Player.STATE_ENDED, Player.STATE_IDLE -> {
                        // Safe: cancel countdown
                        stopTracking()
                    }
                }
            }
        }
    }
}
```

### Blueprint D: Playback Retry Manager (Exponential Backoff)
```kotlin
package com.purefusioniptv.player

import android.os.Handler
import android.os.Looper
import androidx.media3.common.PlaybackException
import androidx.media3.common.Player

class PlaybackRetryManager(
    private val player: Player,
    private val onReloadStream: () -> Unit,
    private val onShowErrorOverlay: (errorCode: Int) -> Unit
) {
    private val handler = Handler(Looper.getMainLooper())
    private var retryCount = 0
    private val maxRetries = 3

    fun handlePlaybackError(error: PlaybackException) {
        val errorCode = error.errorCode
        
        if (retryCount < maxRetries) {
            retryCount++
            // Calculate exponential backoff delay: 1s, 2s, 4s
            val delayMs = (Math.pow(2.0, (retryCount - 1).toDouble()) * 1000).toLong()
            
            handler.postDelayed({
                // Execute silent resync attempt
                onReloadStream()
            }, delayMs)
        } else {
            // All 3 attempts exhausted: Halt player and show "An error occurred: Code XXX. Retrying..."
            onShowErrorOverlay(errorCode)
            
            // Continue persistent background retries every 10 seconds to auto-recover when server goes back online
            handler.postDelayed({
                onReloadStream()
            }, 10000L)
        }
    }

    fun reset() {
        handler.removeCallbacksAndMessages(null)
        retryCount = 0
    }
}
```
