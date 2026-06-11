# TiViMate Catch-up Controls & Timeshift Architecture — Deep Dive

This document provides a comprehensive technical audit and architectural breakdown of **TiViMate’s Catch-up (Timeshift) playback controls, EPG alignment, transport seek overlays, and dynamic URL interpolation engines**. It details how the app bridges the gap between historical EPG data and dynamic streaming files, translates user remote controls (D-pad Left/Right/Up/Down) into absolute epoch seek parameters, and implements a seamless live-to-catchup timeshift transition.

---

## 1. CATCH-UP UI & TRANSPORT CONTROLS (COUCH EXPERIENCE)

Catch-up playback allows the user to browse backward in the TV Guide and watch past programs. TiViMate packages this raw backend capability into a premium, leanback-focused interface:

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│                  Active Catch-up Stream                │
│                                                        │
│                                                        │
│  [ OSD Seek Timeline Overlay ]                         │
│  ◀◀ [10:30 AM] ━━━━━━━⬤━━━━━━━━━━━━━━━━━━━ [11:30 AM]  │
│    Local Time: 10:55 AM (Duration: 60 mins)            │
│    Active EPG: "Sunday Sports Review"                  │
└────────────────────────────────────────────────────────┘
```

### Key Interactive Control Features:
* **The Catch-up Indicator**: Concluded programs in the EPG grid that support archive playback display a distinct "clock" icon. This is dynamically evaluated in the layout using:
  $$\text{Playable} = \text{Program.stopTime} < \text{CurrentTime} \quad \text{AND} \quad \text{Playlist.catchupDays} > 0$$
* **Timeshift Live Pausing**: When a user watches a live channel, tapping the **Pause** button triggers an automatic timeshift session. The player checks if the channel supports catch-up, queries the server timeshift endpoint, switches streams silently, and pauses at the exact live offset.
* **Remappable D-pad Seek Mappings**: Users can map D-pad clicks to seek behaviors in settings:
  * **Short Press Left/Right**: Seek backward/forward by user-configured step intervals (e.g., 10s, 30s, 2 mins).
  * **Long Press Left/Right or RW/FF buttons**: Activates fast seeking. A speed indicator (e.g. `2x`, `4x`, `8x`) updates the playhead calculations.
  * **Up/Down Keys**: Remapped in settings to trigger the OSD seek overlay or display the channel history timeline.

---

## 2. DYNAMIC TIMESHIFT URL RESOLUTION (EPG ENGINE)

IPTV providers structure timeshift archives in different formats. TiViMate parses `#EXTINF` metadata tokens from playlist imports (`catchup="default"`, `catchup-days="7"`, `catchup-source="url"`) and maps them into SQLite tables. 

At playback time, the player controller dynamically resolves the streaming link:

```
[ User Selects EPG Program ] ──► [ Retrieve Channel catchup_type & catchup_source ]
                                                       │
                                   (Construct Timeshift Query URL)
                                                       ▼
  ┌────────────────────────────────────────────────────┼────────────────────────────────────────────────────┐
  ▼ (CatchupType: XTREAM)                              ▼ (CatchupType: FLUSSONIC)                           ▼ (CatchupType: APPEND)
  Splits player URL paths:                             Appends timeshift boundaries:                         Replaces XMLTV tokens:
  Injects `/timeshift/` and path                       `stream-name-{start_epoch}-`                         `catchupSource.replace(${start},`
  parameters (duration/start_time)                     `{duration_seconds}.ts`                              `startEpochSeconds)`
```

### The 3 Core Catch-up Formats:

#### A. Xtream Codes Timeshift Format
Xtream Codes processes catch-up by altering the URL path structure. The normal live stream URL:
`http://domain:port/live/username/password/stream_id.ts`
is dynamically converted into:
`http://domain:port/timeshift/username/password/duration_minutes/start_time_formatted/stream_id.ts`

* **Start Time Format**: `YYYY-MM-DD-HH-MM` (e.g. `2026-06-04-16-30`).
* **Duration**: The length of the program in minutes.

#### B. Flussonic Timeshift Format
Flussonic indexes timeshift archives directly inside the file name using Unix Epoch timestamps:
`http://domain:port/stream/mono-timeshift-{start_epoch}-{duration_seconds}.ts`

#### C. Append / Tokenized Format
For standard M3U playlists, the provider supplies a template query format in the `catchup-source` tag. The tokenizer dynamically replaces placeholders:
* `${start}`: Program start time in Unix Epoch seconds.
* `${end}`: Program stop time in Unix Epoch seconds.
* `${duration}`: Program duration in seconds.
* `${offset}`: Time offset parameters.

---

## 3. SEAMLESS STATE SEEK RUNTIME (HOT-SWAPPING)

Many timeshift streams are processed as dynamic HLS/MPEG-TS files. Because these servers do not support standard HTTP Byte Range queries (which allow seeking inside a static file), standard media players crash or stall when seeking catch-up streams.

TiViMate bypasses this using an **Active State Reload & Hot-Swapping** mechanism:

```
[ User Drags Seek Slider to Position T ]
                   │
                   ▼
[ Calculate absolute target Unix Epoch time: startEpochSeconds + T ]
                   │
                   ▼
[ Re-interpolate dynamic Catch-up URL using the new Start Time ]
                   │
                   ▼
[ Silent DataSource Prepare on active ExoPlayer instance ]
                   │
                   ▼
[ Player connects to new socket & resumes video decoding seamlessly ]
```

1. **Epoch Time Calculation**: When the user drags the timeline seek bar, the controller computes the target timestamp:
   $$\text{Target Epoch} = \text{Program.startTimeSeconds} + \text{TimelineSeekPosition}$$
2. **Dynamic URL Re-generation**: The catch-up URL resolver generates a new streaming URL using the computed $\text{Target Epoch}$ as the new start parameter.
3. **ExoPlayer Hot-Swapping**: The controller disposes of the old socket connection, updates the active `MediaItem` URL parameter inside ExoPlayer, calls `player.prepare()`, and resumes decoding from the new playhead. This is executed with a smooth, translucent loading overlay to hide the reload gap.

---

## 4. INTEGRITY PROTECTION & TIME CALIBRATION

Catch-up streaming is highly sensitive to clock discrepancies. If not protected, it will result in streaming errors (`HTTP 404 Not Found`) or account bans.

### I. Timezone Offset Stacking
EPG files use Unix Epoch or local server formats. If the IPTV provider's server is in timezone UTC+2 and the TV box is in UTC-4, requests for catch-up using local box timestamps will fail.
TiViMate implements a triple-stackedtimezone resolution query:
```sql
SELECT p.tvg_shift, p.server_timezone, c.user_catchup_time_offset FROM channels c...
```
* **`tvg_shift`**: The default provider timezone shift index.
* **`server_timezone`**: Detected from the Xtream Codes server payload.
* **`user_catchup_time_offset`**: A manual offset configuration (allow offsets +/- 12 hours) that the user can set in preferences to calibrate clock drift.

### II. Seek Throttling (Banning Protection)
IPTV servers are highly restrictive. If a user rapidly taps Left/Right on a remote to seek, it will trigger multiple back-to-back HTTP requests. IPTV panels will flag this as a DDoS attempt and **blacklist the user's IP/MAC address**.
* **The Safeguard**: TiViMate debounces seek inputs. When a user drags the progress bar or clicks seeking buttons, the player updates the visual UI playhead instantly, but **delays the actual HTTP stream request for 1.5 seconds**. If another seek input is received within 1.5 seconds, the countdown resets, executing only a single HTTP connection once the user releases the remote button.

### III. System Time Calibration
If the Android TV box system time drifts (which frequently occurs when devices are disconnected from NTP servers on startup), catch-up files cannot be located.
* **The Safeguard**: During app startup, TiViMate's network client performs a quick `HEAD` call to the provider server. It extracts the HTTP `Date` header, compares it to the local system time, and calculates a global calibration delta ($\Delta_t$) applied to all subsequent timeshift epoch queries.

---

## 5. ACTIONABLE TIMESHIFT IMPLEMENTATION BLUEPRINTS

Use these blueprints to implement catch-up controls and seek actions in your TV player.

### Blueprint A: Scoped Catch-up Overrides (SQLite Schema)
```kotlin
package com.purefusioniptv.data.db.entities

import androidx.room.ColumnInfo
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "playlists")
data class PlaylistEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val url: String,
    @ColumnInfo(name = "catchup_type") val catchupType: Int, // 0 = None, 1 = Xtream, 2 = Flussonic, 3 = Append
    @ColumnInfo(name = "catchup_hours") val catchupHours: Int?,
    @ColumnInfo(name = "catchup_source") val catchupSource: String?,
    @ColumnInfo(name = "user_catchup_type") val userCatchupType: Int?,
    @ColumnInfo(name = "user_catchup_hours") val userCatchupHours: Int?
)

@Entity(tableName = "channels")
data class ChannelEntity(
    @PrimaryKey val id: Long,
    @ColumnInfo(name = "playlist_id") val playlistId: Long,
    val name: String,
    @ColumnInfo(name = "catchup_hours") val catchupHours: Int?,
    @ColumnInfo(name = "user_catchup_time_offset") val userCatchupTimeOffset: Int? // Offset in seconds
)
```

### Blueprint B: Polymorphic Catch-up URL Builder
```kotlin
package com.purefusioniptv.player

import android.net.Uri
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.TimeZone

object CatchupUrlBuilder {

    fun buildCatchupUrl(
        channelUrl: String,
        catchupType: Int,
        catchupSource: String?,
        programStartEpoch: Long,
        programDurationSeconds: Long,
        userTimeOffsetSeconds: Long = 0
    ): String {
        val adjustedStart = programStartEpoch + userTimeOffsetSeconds

        return when (catchupType) {
            1 -> { // Xtream Codes Timeshift Format
                val durationMinutes = programDurationSeconds / 60
                val formattedTime = formatXtreamTimestamp(adjustedStart)
                interpolateXtreamUrl(channelUrl, durationMinutes, formattedTime)
            }
            2 -> { // Flussonic Format
                val cleanUrl = channelUrl.substringBeforeLast(".ts")
                "$cleanUrl-${adjustedStart}-${programDurationSeconds}.ts"
            }
            3 -> { // Append Tokenizer Format
                if (catchupSource.isNullOrEmpty()) return channelUrl
                catchupSource
                    .replace("\${start}", adjustedStart.toString())
                    .replace("\${duration}", programDurationSeconds.toString())
                    .replace("\${offset}", (userTimeOffsetSeconds / 60).toString())
            }
            else -> channelUrl
        }
    }

    private fun formatXtreamTimestamp(epochSeconds: Long): String {
        val sdf = SimpleDateFormat("yyyy-MM-dd-HH-mm", Locale.US)
        sdf.timeZone = TimeZone.getTimeZone("UTC")
        return sdf.format(Date(epochSeconds * 1000L))
    }

    private fun interpolateXtreamUrl(baseUrl: String, durationMins: Long, formattedTime: String): String {
        val uri = Uri.parse(baseUrl)
        val pathSegments = uri.pathSegments
        if (pathSegments.size < 4) return baseUrl

        // Transform: /live/user/pass/stream_id.ts -> /timeshift/user/pass/duration/time/stream_id.ts
        val username = pathSegments[1]
        val password = pathSegments[2]
        val streamId = pathSegments[3]

        return "${uri.scheme}://${uri.host}:${uri.port}/timeshift/$username/$password/$durationMins/$formattedTime/$streamId"
    }
}
```

### Blueprint C: Debounced Seek Dispatcher (Account Ban Protection)
```kotlin
package com.purefusioniptv.player

import android.os.Handler
import android.os.Looper

class DebouncedSeekController(
    private val onSeekTriggered: (targetEpochSeconds: Long) -> Unit,
    private val debounceDelayMs: Long = 1500L
) {
    private val handler = Handler(Looper.getMainLooper())
    private var pendingTargetEpoch: Long = -1L

    private val seekRunnable = Runnable {
        if (pendingTargetEpoch != -1L) {
            onSeekTriggered(pendingTargetEpoch)
            pendingTargetEpoch = -1L
        }
    }

    /**
     * Call this whenever the user performs a seek input (D-pad Left/Right or bar drag).
     * Updates the local variable instantly and triggers the connection only after debounce timeout.
     */
    fun postSeekRequest(targetEpochSeconds: Long) {
        pendingTargetEpoch = targetEpochSeconds
        handler.removeCallbacks(seekRunnable)
        handler.postDelayed(seekRunnable, debounceDelayMs)
    }

    fun cancelPendingSeeks() {
        handler.removeCallbacks(seekRunnable)
        pendingTargetEpoch = -1L
    }
}
```
