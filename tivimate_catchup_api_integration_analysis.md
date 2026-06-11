# TiViMate Catch-up & Portal API Client — Technical Architecture Analysis

This document provides a highly detailed architectural analysis of **TiViMate’s Catch-up (Timeshift)** playback engine and its **Multi-Portal API Client (Xtream Codes & Stalker)** integration. It dissects how the application dynamically constructs catch-up URLs, manages state machine seek requests on live servers, and handles token auto-refresh routines for Stalker portals. It provides concrete blueprints to replicate these systems in **PureFusionIPTV**.

---

## 1. USER-FLOW LEVEL BEHAVIOR

* **Historical Program Playback**: While browsing the TV Guide, the user can scroll left in time to see past EPG programs. A small "circular clock" or "catch-up" icon appears on programs that have already concluded. Tapping D-pad Center on a historical program instantly starts playback of that broadcast.
* **Full Transport Seek Control**: During catch-up playback, the user has a standard timeline bar and can seek back and forth (Rewind / Fast-Forward) exactly as if they were watching a local recording.
* **Seamless Live-to-Catchup Transition**: If the user is watching a live channel, they can pause playback. The stream instantly shifts to the catch-up server, permitting active pause/resume.
* **Automatic Authentication Recovery**: If a stream link expires during a long playback session, the app silently re-authenticates with the server in the background and resumes playback without throwing error modals or forcing the user back to the menu.

---

## 2. ARCHITECTURAL PATTERNS UNDER THE HOOD

IPTV catch-up is highly fragmented. Different IPTV middleware solutions (Xtream Codes, Flussonic, Stalker, Ministra) use completely different API endpoints and query tokens. TiViMate manages this complexity through a unified **Catch-up Resolver** and a stateful **API Session Broker**.

```mermaid
graph TD
    A[EPG Catchup Click] --> B{Resolve Playlist Catchup Type}
    
    B -->|Xtream Codes| C[Xtream Timeshift Interpolator]
    B -->|Flussonic / Append| D[M3U Token Parser]
    B -->|Stalker Portal| E[Stalker Action Client]
    
    C -->|Format: Duration & Start Seconds| F[ExoPlayer HTTP Source]
    D -->|Replace: ${start} / ${duration}| F
    E -->|API Handshake: /portal.php| G{Verify Stalker Token}
    
    G -->|Valid| H[Fetch Action Link] --> F
    G -->|Expired / 403| I[Re-Authenticate MAC/Token] --> G
    
    style B fill:#1a365d,stroke:#3182ce,stroke-width:2px,color:#fff
    style G fill:#2c3e50,stroke:#e74c3c,stroke-width:2px,color:#fff
    style F fill:#27ae60,stroke:#2ecc71,stroke-width:2px,color:#fff
```

### The 3 Core Architectural Pillars

1. **Polymorphic Catch-up URL Builder**:
   Rather than hardcoding a single streaming format, the app reads EPG/Playlist columns (`catchup_type`, `catchup_source`) and tokenizes the target URL on-the-fly. It supports standard formats:
   * **Xtream Codes (Timeshift)**:
     `http://server.com/timeshift/username/password/duration_minutes/start_time_formatted/stream_id.ts`
   * **Flussonic Timeshift**:
     `http://server.com/stream/mono-timeshift-{start_epoch}-{duration}.ts`
   * **Append Tokenizer**:
     Replaces tokens `${start}` (Unix Epoch), `${end}`, `${duration}` (seconds), and `${offset}` in standard M3U streams.
2. **Stalker Token Lifecycle Broker**:
   Stalker portals rely on MAC-address-locked sessions that generate transient playback links via:
   `action=create_link&cmd=ffmpeg+http://localhost/ch_url`
   TiViMate stores the active token (`stalker_token`) transactionally in SQLite. If ExoPlayer catches a `HttpDataSourceException` with a `403 Forbidden` error, the session manager intercepts it, triggers a background handshake refresh, updates `stalker_token` in `playlists`, and re-routes the active ExoPlayer instance to the newly generated URL without halting the media engine.
3. **Seek-State Reload Mapping**:
   Many timeshift servers do not support standard HTTP Byte Range seeking. When the user drags the player seek bar, the application:
   * Translates the seek timeline position to an **absolute Unix Epoch Timestamp**.
   * Re-interpolates the target catchup query URL with the new timestamp.
   * Silently prepares and swaps the ExoPlayer DataSource, resuming video decoding from the selected offset.

---

## 3. CLUE SOURCE (CLASSES & FILES)

* **`sources/p240/C3210.java`** (Playlist/Channel data mapping queries):
  * **Master EPG Joint Load Query**:
    Extracts catchup details and user overrides dynamically during guide rendering:
    ```sql
    SELECT m.id, m.channel_id, c.name channel_name, c.custom_name custom_channel_name,
           p.catchup_type playlist_catchup_type, p.catchup_hours playlist_catchup_hours,
           p.user_catchup_type playlist_user_catchup_type, p.user_catchup_hours playlist_user_catchup_hours, 
           c.catchup_hours
    FROM my_programs m
    LEFT OUTER JOIN channels c ON m.channel_id == c.id
    LEFT OUTER JOIN playlists p ON c.playlist_id == p.id
    ```
    If a user manually adjusts catch-up offsets in preferences, `user_catchup_time_offset` (line 86) is applied onto the stream arguments to calibrate timezone differences.
* **`sources/p240/C3220.java`** (Token Persistence):
  * **Case 9 (Update transient Stalker token in SQLite)**:
    ```sql
    UPDATE playlists SET stalker_token = ? WHERE id == ?
    ```
    This secures active portal authentication across restarts, avoiding costly handshakes on every single channel change.
* **`sources/p047/C1367.java`** (Playlist Parser):
  * Parses `#EXTINF` metadata attributes: `catchup="default"`, `catchup-source="url"`, and `catchup-days="7"`, mapping them directly to database playlist structures.

---

## 4. CONCEPTUAL REPLICATIONS FOR PUREFUSIONIPTV

Use this complete blueprint to implement catch-up url resolving and token updates in **PureFusionIPTV**.

### Step 1: The Catch-up URL Resolving Subsystem

Write a flexible Kotlin utility to format timeshift URLs based on playlist configuration.

```kotlin
enum class CatchupType {
    DEFAULT,
    XTREAM,
    FLUSSONIC,
    APPEND
}

class CatchupUrlResolver {
    
    fun resolveCatchupUrl(
        baseUrl: String,
        catchupType: CatchupType,
        catchupSource: String?,
        programStartEpochSeconds: Long,
        programDurationSeconds: Long,
        timeOffsetSeconds: Long = 0
    ): String {
        // Adjust for timezone differences
        val startTime = programStartEpochSeconds + timeOffsetSeconds
        
        return when (catchupType) {
            CatchupType.XTREAM -> {
                // Xtream Codes timeshift format:
                // url/timeshift/username/password/duration_mins/YYYY-MM-DD-HH-MM/stream_id.ts
                val durationMinutes = programDurationSeconds / 60
                val formattedTime = formatXtreamTime(startTime)
                buildXtreamTimeshiftUrl(baseUrl, durationMinutes, formattedTime)
            }
            CatchupType.FLUSSONIC -> {
                // Flussonic format: baseUrl-{start}-{duration}.ts
                baseUrl.replace(".ts", "") + "-${startTime}-${programDurationSeconds}.ts"
            }
            CatchupType.APPEND -> {
                if (catchupSource.isNullOrEmpty()) return baseUrl
                // Replace standard XMLTV tokens in the catchup source string
                catchupSource
                    .replace("\${start}", startTime.toString())
                    .replace("\${duration}", programDurationSeconds.toString())
                    .replace("\${offset}", (timeOffsetSeconds / 60).toString())
            }
            else -> baseUrl
        }
    }
    
    private fun formatXtreamTime(epochSeconds: Long): String {
        val sdf = java.text.SimpleDateFormat("yyyy-MM-dd-HH-mm", java.util.Locale.US)
        sdf.timeZone = java.util.TimeZone.getTimeZone("UTC")
        return sdf.format(java.util.Date(epochSeconds * 1000L))
    }
    
    private fun buildXtreamTimeshiftUrl(baseUrl: String, durationMins: Long, formattedTime: String): String {
        // Splitting and re-injecting /timeshift/ segment into standard player url
        // http://domain:port/live/user/pass/stream_id.ts -> http://domain:port/timeshift/user/pass/duration/time/stream_id.ts
        val uri = android.net.Uri.parse(baseUrl)
        val pathSegments = uri.pathSegments
        if (pathSegments.size < 4) return baseUrl
        
        val username = pathSegments[1]
        val password = pathSegments[2]
        val streamId = pathSegments[3]
        
        return "${uri.scheme}://${uri.host}:${uri.port}/timeshift/$username/$password/$durationMins/$formattedTime/$streamId"
    }
}
```

### Step 2: Resilient Stalker Portal Handshake Handler

Implement a session client that automatically recovers Stalker tokens upon expiry.

```kotlin
class StalkerSessionBroker(
    private val apiService: StalkerApiService,
    private val playlistDao: PlaylistDao
) {
    suspend fun getValidStreamUrl(
        playlistId: Long, 
        macAddress: String, 
        portalUrl: String, 
        channelCmd: String
    ): String = withContext(Dispatchers.IO) {
        val playlist = playlistDao.getPlaylistById(playlistId) 
            ?: throw IllegalArgumentException("Playlist not found")
            
        var activeToken = playlist.stalkerToken
        
        if (activeToken.isNullOrEmpty()) {
            activeToken = executeHandshake(portalUrl, macAddress, playlistId)
        }
        
        try {
            // Attempt to get stream link
            val linkResponse = apiService.createLink(portalUrl, activeToken, channelCmd)
            if (linkResponse.isSuccessful && linkResponse.body()?.url != null) {
                return@withContext linkResponse.body()!!.url
            }
        } catch (e: Exception) {
            Log.e("StalkerSession", "Stream link fetch failed, refreshing credentials", e)
        }
        
        // Force refresh handshake on error or 403
        val freshToken = executeHandshake(portalUrl, macAddress, playlistId)
        val freshLink = apiService.createLink(portalUrl, freshToken, channelCmd).body()
            ?: throw IOException("Failed to get Stalker stream link after token refresh")
            
        return@withContext freshLink.url
    }
    
    private suspend fun executeHandshake(portalUrl: String, macAddress: String, playlistId: Long): String {
        val handshakeResponse = apiService.handshake(portalUrl, macAddress)
        val token = handshakeResponse.body()?.token 
            ?: throw IOException("Stalker Handshake rejected MAC address")
            
        playlistDao.updateStalkerToken(playlistId, token)
        return token
    }
}
```

---

## 5. DIRECT COPYING WARNINGS (DO NOT COPY)

* **`p240.C3210` Column Identifiers**: Do not directly copy variables or hardcoded column index shifts like `iM75427 = AbstractC3586.m7546(..., "catchup_type")`. These represent obfuscated Room helpers compiled by R8. Let Room handle column mapping automatically by using standard Kotlin fields mapped directly via compile-time reflection.
* **Hardcoded Regex Matching on URLs**: Avoid parsing Xtream URLs using custom fragile regex filters. Always use Android's structured `Uri.parse(url)` to extract path parameters and prevent crashes on domain names that contain custom characters or ports.

---

## 6. IMPLEMENTATION RISKS

* **Clock Out-of-Sync Buffering**:
  Timeshift servers require high timestamp precision. If the local TV box clock drifts by even 1 minute relative to the IPTV provider's stream server clock, requests for the timeshift stream will return `404 Not Found` or play a different program entirely.
  * *Mitigation*: **Server-Time Syncing**. Perform a quick HTTP HEAD request to the provider server on app startup, check the returned `Date` header, and calculate a global clock drift offset variable to calibrate catch-up query parameters.
* **API Rate Limiting & Account Bans**:
  Aggressive seeking on catchup streams causes multiple rapid HTTP calls. Stalker portals will trigger automated DDoS protection and temporarily block the user's MAC address if they receive 5 token requests in under 10 seconds.
  * *Mitigation*: **Seek Throttle Debouncing**. Restrict EPG clicks and seek actions using a debounce filter. Allow a minimum of **1.5 seconds** between consecutive stream load commands to safeguard connections.
