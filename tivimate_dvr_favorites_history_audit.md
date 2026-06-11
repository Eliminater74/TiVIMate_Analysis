# TiViMate DVR, Favorites & Playback History — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s background PVR/DVR engine, double-buffered favorites indexing, and smart history queue**. It dissects how the application wakes the CPU from deep sleep states for scheduled recordings, structures virtual category joins for favorites, and filters quick-zapping noise from history lists using a surfing delay buffer.

---

## 1. ADVANCED PVR / DVR ENGINE & BACKGROUND COLD-WAKE

On modern Android TV devices, background services are aggressively restricted to save battery life. If the TV screen is shut off, the device enters deep sleep (CEC standby), suspending standard timers and killing unmanaged threads.

```
 [ AlarmManager (OS) ] ─── Exact Wake ───► [ PvrBroadcastReceiver ]
                                                   │
                         (Spawns persistent Foreground Service)
                                                   ▼
                                      [ PvrForegroundService ]
                                                   │
                            (Acquires Strong PARTIAL_WAKE_LOCK)
                                                   ▼
                                         [ OkHttp Stream Pipe ]
                                                   │
                            (Direct bytes dump to 128KB RAM buffer)
                                                   ▼
                                        [ FileOutputStream / SMB ]
```

### The Background Wake-up Routine

1.  **Exact Scheduling**:
    When a user schedules an EPG recording, the app does not rely on WorkManager (which the OS can defer for hours). It schedules an exact event using Android's `AlarmManager` with `setExactAndAllowWhileIdle()` or `setAlarmClock()`.
2.  **Broadcast Receiver & WakeLocks**:
    At the program's exact start time, the OS wakes up and triggers a `BroadcastReceiver`. The receiver immediately:
    *   Acquires a CPU `PowerManager.WakeLock` (`PARTIAL_WAKE_LOCK`) with a hardcoded fail-safe timeout (e.g. 2 hours). This keeps the TV CPU active even when the display panel remains off.
    *   Launches a persistent **Foreground Service** (`ar.tvplayer.core.pvr.PvrService`). Spawning a Foreground Service elevates the process priority, protecting it from being killed under low-memory conditions.
3.  **Low-Footprint Direct Stream Piping**:
    Initializing a full `ExoPlayer` instance to record would consume massive CPU and memory. The `PvrService` completely bypasses the media player. It uses a clean `OkHttpClient` socket pipe to fetch raw TS bytes directly into a **128KB RAM buffer**, dumping them directly onto the output file stream:
    ```kotlin
    val buffer = ByteArray(128 * 1024) // 128KB buffer
    var bytesRead: Int
    while ((bytesRead = inputStream.read(buffer)) != -1) {
        outputStream.write(buffer, 0, bytesRead)
    }
    ```
4.  **Storage Supervisor Watchdog**:
    To prevent the TV from locking up due to storage exhaustion, a background daemon checks available disk partition bytes:
    ```kotlin
    val stats = StatFs(recordingDir.absolutePath)
    val availableBytes = stats.availableBlocksLong * stats.blockSizeLong
    if (availableBytes < 500 * 1024 * 1024L) { // 500MB safety threshold
        stopRecordingGracefully(lowSpace = true)
    }
    ```

---

## 2. DOUBLE-BUFFERED FAVORITES INDEXING

Duplicating channel records in SQLite to represent a "Favorites" list causes database sync chaos. TiViMate keeps favorites clean and relational through virtual joins.

### The Virtual Favorites Category
In TiViMate's sidebar, the "Favorites" category is a virtual folder. It does not exist in the M3U playlist file. It is resolved dynamically using a single `is_favorite` integer flag in the `channels` table:

```sql
SELECT c.id, c.playlist_id, p.name playlist_name, c.category_id, 
       c.name, c.logo, c.url, c.is_favorite
FROM channels c
INNER JOIN playlists p ON c.playlist_id == p.id
WHERE c.is_favorite == 1 
  AND c.deleted_time IS NULL
  AND p.is_enabled == 1
ORDER BY c.position_in_favorites ASC
```

By querying this across all playlists, the TV guide merges and updates favorited streams in real-time, completely preventing file duplication.

---

## 3. PLAYBACK HISTORY & THE SURFING DELAY FILTER

If a user is actively channel-surfing (switching channels every 2 seconds to see what is playing), adding every single channel to their history list creates a cluttered queue of un-watched streams. 

TiViMate solves this through a stateful **Surfing Delay Filter**.

```
[ Channel Selected ] ──► [ Cancel Active surfJob ]
                               │
                               ▼ (Trigger Coroutine Delay)
                        [ delay(5000) ] (5-Second Surfing Filter)
                               │
                               ├─► IF User Zaps Away ──► Canceled (No History Entry)
                               │
                               └─► IF Time Elapses   ──► Commit to Database
                                                         * UPDATE channels SET last_turn_on_time = ?
                                                         * Pull active EPG Snapshot & push to Queue
```

### The 5-Second History Delay Logic

1.  **Dynamic Job Cancellation**:
    The history manager maintains an active Coroutine `Job` reference (`surfJob`). When a channel change is detected, the manager immediately calls `surfJob?.cancel()`.
2.  **The 5-Second Filter**:
    A new coroutine is launched on the main thread:
    ```kotlin
    var surfJob: Job? = null

    fun onChannelTuned(channelId: Long) {
        surfJob?.cancel() // Cancel previous pending history write
        surfJob = CoroutineScope(Dispatchers.Main).launch {
            delay(5000) // Wait for a stable 5-second viewing period
            commitChannelToHistory(channelId)
        }
    }
    ```
    *   **If User Zaps Away**: The job is canceled before the timer finishes, leaving their history completely clean of short-surfing clutter.
    *   **If User Stays**: The timer concludes, and the channel is committed to the SQLite history table by updating `last_turn_on_time` (`C3210.java` line 415).
3.  **Active EPG Snapshots**:
    Instead of writing static metadata to the history queue, each history record holds only a `channel_id` and `timestamp`. When displaying the "Recent Channels" overlay, the UI dynamically joins the EPG table (`my_programs` or `tvg_programs`) to fetch the **live EPG snapshot** on-the-fly, ensuring the program titles in history are always current rather than displaying stale strings from hours ago.
