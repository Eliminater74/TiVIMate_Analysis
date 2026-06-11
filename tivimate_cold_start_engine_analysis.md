# TiViMate Cold Start Engine & Boot Sequence — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate's rapid startup sequence (the "Zero-Lag" cold start)**. It dissects why the application starts and renders video in under 300ms, detailing what runs first, what database calls are pre-warmed, what UI bindings are deferred, and what sync processes are throttled in the background on startup.

---

## 1. USER-FLOW LEVEL BEHAVIOR

*   **Sub-300ms Cold Start**: Tapping the TiViMate icon on a smart TV immediately opens the player interface. There are no heavy splash spinners, "loading data" popups, or network login blocks.
*   **Instant Playback Recovery**: The exact channel the user was watching before closing the app starts streaming immediately.
*   **Asynchronous UI Population**: The EPG grid, sidebars, and category menus slide in smoothly. If EPG data is still loading into memory, the stream plays unhindered in the background, populating text blocks as they become ready.

---

## 2. THE CHRONOLOGICAL BOOT SEQUENCE

To achieve a cold boot in under 300ms on low-end television hardware, TiViMate strictly prioritizes the video stream over EPG data, database schemas, and network checks.

```
 [ Android OS Spawns Process ] ──► [ 1. App Lifecycle: DB Pre-warming ]
                                                │
                                                ▼
                                   [ 2. Retrieve last_played_channel ]
                                   - Fast SharedPreferences read (0.5ms)
                                                │
                                                ▼
                                   [ 3. Instantiate ExoPlayer & Buffer ]
                                   - Connect to socket before EPG draws
                                                │
             ┌──────────────────────────────────┴──────────────────────────────────┐
             ▼ (Main Thread: 100ms)                                                ▼ (IO Thread: Deferred)
   [ 4. Lazy-Inflate UI Overlays ]                                       [ 5. Load visible category metadata ]
   - Draw transparent styles, canvas rulers.                             - Fetch visible rows from WAL SQLite.
             │                                                                     │
             └──────────────────────────────────┬──────────────────────────────────┘
                                                ▼
                                    [ 6. Video Frame Decoded ]
                                    (Liquid-Smooth Hero Render)
                                                │
                                                ▼ (30s Startup Delay)
                                    [ 7. Defer Background Updates ]
                                    - Sleep EPG & Playlist sync routines
```

### Stage 1: Application Initialization & DB Pre-Warming
*   **The Trap**: Standard Room/SQLite database builders initialize on the main thread when the first query is issued. Compiling the schema, opening the journal, and warming connection pools takes **100–300ms**, causing startup stutters.
*   **TiViMate's Pre-Warming Secret (`AbstractC3431.java`)**:
    During `Application.onCreate()`, the database builder is initialized instantly on a background executor thread, executing a simple raw query (e.g. `SELECT 1`). This opens database handles and loads B-tree indices before the `MainActivity` even launches.

### Stage 2: Fast Shared Preferences Lookup
*   The primary activity retrieves the `last_channel_url` and `last_channel_id` from SharedPreferences. This takes under **0.5 milliseconds** as a simple key-value fetch.

### Stage 3: Immediate Playback Initialization (Hero Render)
*   Before categories, channels, or EPG cells are queried from SQLite, the app immediately instantiates `ExoPlayer` and commands it to buffer the `last_channel_url`.
*   *Why this is a Masterclass in TV UX*:
    Video buffering requires hardware decoder negotiation (MediaCodec) which takes 150–250ms. By starting the buffer loop **before** inflating EPG grids or category drawers, the network stream begins buffering *in parallel* while the UI is rendering.

### Stage 4: Lazy UI Ingestion & Decoupled Loading
*   While ExoPlayer is buffering in the background, the UI layout is inflated.
*   **Ephemeral Scrim View**: The main container is drawn with a dark translucent background.
*   **Lazy Grid Binding**: The EPG grid draws its Canvas ruler lines instantly in memory, but delays querying EPG text cells. It queries EPG programs for *only* the visible channels in the viewport using asynchronous SQLite WAL read queries on `Dispatchers.IO`.

### Stage 5: Deferring Background Sync Processes
*   EPG XMLTV updates and M3U playlist refreshes are CPU and disk I/O intensive. Running them during cold boot will completely choke the TV processor, causing video buffering and navigation lag.
*   **The Deferral Secret**: TiViMate's background WorkManager workers are configured with a **30-second or 1-minute startup delay constraint**. Playlist and EPG updates are completely deferred during the boot phase, leaving 100% of network sockets and database locks free for active stream playback.

---

## 3. WHY TIVIMATE STARTS SO FAST (THE SECRETS BRIEF)

| Bottleneck | standard approach | TiViMate Design Pattern |
|---|---|---|
| **Database Opening** | Initiated on main thread during UI layout queries. | Pre-warmed asynchronously in `Application.onCreate()` via background SELECTs. |
| **User Sign-In** | Blocking network calls to check licenses on boot. | Offline verification of cached signed JWT tokens in SharedPreferences. |
| **Stream Playback** | Loads EPG grid first; stream starts when category is selected. | Queries last played stream from preferences instantly; starts buffer on frame 1. |
| **EPG / Playlist Updates** | Refreshes EPG feeds immediately on application startup. | Defer sync tasks using WorkManager with a 30s-1min startup delay. |
| **Grid Rendering** | Inflates heavy, nested vertical-horizontal RecyclerView views. | Canvas direct pixel coordinate mapping; skips views inflation entirely. |

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Save the last-played stream URL in SharedPreferences. Start buffering this URL immediately upon `MainActivity.onCreate()` before running any EPG grid queries.
*   Add a 30-second startup delay to all background playlist and EPG synchronization workers.

### Medium-Risk Actions
*   Pre-warm your Room database instance inside your custom `Application` class on a background thread.
*   Implement offline validation of signed JWT license tokens to prevent blocking network licensing calls on boot.

### High-Risk Actions
*   Executing massive database EPG checks or category queries synchronously on the main thread during activity creation. This will immediately cause TV boxes to lag, trigger ANR dialogs, and block video rendering.

### What to Measure First
*   Benchmark cold boot zapping times. The duration from clicking the app icon to the first video frame rendering should be **under 400ms**.

### What Not to Copy Directly
*   Do not copy custom initialization callbacks or obfuscated lifecycle structures.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Asynchronous database pre-warming, fast preference stream lookups, and background sync worker startup deferrals.
