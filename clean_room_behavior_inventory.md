# TiViMate Clean-Room Behavior Inventory for PureFusionIPTV

This document compiles clean-room behavioral rules, UX patterns, and architectural designs extracted from the analysis of premium TV-centric media player paradigms (specifically TiViMate). It acts as a reference blueprint to help **PureFusionIPTV** achieve optimal zapping speeds, EPG reliability, timeshift seek overlays, key mapping behaviors, settings hierarchies, and provider safety rules.

---

## 1. Sources Reviewed

### I. Architectural Reports (Root Directory)
The following documentation reference files inside `i:\GITHUB\Projects\TiViMate-Latest-decompiled` were reviewed and analyzed for architectural lessons:
*   [tivimate_guide_performance_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_guide_performance_analysis.md): Details the 3 Pillars of EPG speed (zero DB I/O during scrolling, Custom Canvas rendering, multi-tier cache), separated SQLite tables (`my_programs` and `tvg_programs`), Unix Epoch Seconds storage, length-based queries, FTS search, and partial XMLTV parsing.
*   [tivimate_channel_change_osd_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_channel_change_osd_analysis.md): Details the channel zapping lifecycle (OkHttp socket pre-warming, LoadControl buffer thresholds), SurfaceView hardware transparency layering, Info Bar snap overlays, D-pad Center layout shifts, and customizable Quick Actions.
*   [tivimate_catchup_api_integration_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_catchup_api_integration_analysis.md): Details historical playbacks, Xtream Codes timeshift path resolution, Flussonic timeshift URLs, tokenized M3U templates, live pause-to-timeshift, seek throttling, and MAC/token silent handshakes.
*   [tivimate_catchup_controls_architecture.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_catchup_controls_architecture.md): Details OSD seek timeline overlay structures, remappable D-pad seek controls, polymorphic timeshift url builders, seek debouncing (1.5s delay), and timezone calibration offset stacks.
*   [tivimate_streaming_freeze_resync_deep_dive.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_streaming_freeze_resync_deep_dive.md): Details the stuck buffer / watchdog freeze detection (5,000ms timer), shared OkHttp connection pool, auto-retry exponential backoff loops (1s, 2s, 4s delay), and software decoder fallbacks for MediaCodec exhaustion.
*   [tivimate_playlist_update_safety_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_update_safety_analysis.md): Details the incremental sync diffing engine, atomic transactions, user configuration column safeguards, and silent fault-tolerant load controls.
*   [tivimate_epg_sync_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_epg_sync_analysis.md): Details the Room-backed WorkManager sync worker, WAL mode database integration, and constraints management.
*   [tivimate_playlist_parser_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_parser_analysis.md): Details streaming parser optimizations using buffered line readers and metadata queue lists.
*   [tivimate_remote_input_menu_architecture.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_remote_input_menu_architecture.md): Details key event routing at the window level, long-press handlers, and dynamic serializable menu actions.
*   [tivimate_playback_controls_settings_audit.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playback_controls_settings_audit.md): Details aspect-ratio measure overrides, multi-track audio overrides, and sliding drawer card widgets.
*   [tivimate_usability_insights_guide.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_usability_insights_guide.md) & [tivimate_design_ux_reference_guide.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_design_ux_reference_guide.md): Outline usability principles, typography rules, color pallets, animations, and visual benchmarks.

### II. Decompiled Code Base (Inspected Folders/Files)
The following actual decompiled source code files under `sources/` were inspected to verify behaviors:
*   [sources/p240/C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java): Audited Room database mapper query constants for playlists, VOD counts, user user-catchup offsets, and EPG program column selections.
*   [sources/p240/C3214.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java): Audited database transaction handlers for WorkManager task parameters and movie search FT4 queries.
*   [sources/p240/C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java): Audited SQLite update queries for playlist names, user agents, Stalker tokens, and `tvg_programs_fts` full-text search matchers.
*   [sources/p363/WindowCallbackC4401.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p363/WindowCallbackC4401.java): Audited top-level `Window.Callback` implementation intercepting `dispatchKeyEvent()` globally before dispatching to lower-level layout controls.
*   [sources/p312/C3860.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p312/C3860.java): Audited the player overlay frame containing ExoPlayer controller hooks, mapping D-pad left/right seeks, fast-forward bounds, aspect ratio zooms, track selection popup panels, and fading timer loops.
*   [sources/p127/C2162.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p127/C2162.java): Audited series parsers for Xtream Codes and Stalker metadata parameters.

---

## 2. Executive Summary

The core clean-room findings for PureFusionIPTV are summarized into five actionable architectural principles:
1.  **Zapping Velocity**: Sub-1.5s channel changes are achieved by sharing a singleton OkHttpClient socket connection pool across stream loads, setting buffering thresholds aggressively (1s minimum, 2.5s maximum, 500ms playback start), and avoiding decoder restarts on aspect ratio adjustments.
2.  **Watchdog Stabilization**: Video freeze conditions are resolved by running a 5-second countdown timer whenever ExoPlayer transitions to `STATE_BUFFERING`. If the timer expires or fails to render a first frame within 5 seconds of the `STATE_READY` state, a silent resync tears down the socket, requests a fresh token/link, and re-prepares the player.
3.  **Direct Surface Rendering**: For a liquid-smooth 60fps EPG guide grid, PureFusionIPTV must bypass heavy nested RecyclerView layouts. The guide loads all visible-window programs into a sorted RAM list once, binary-searches items on-the-fly, and draws cells directly onto a hardware-accelerated Canvas.
4.  **Centralized Key Interception**: Remote control inputs are intercepted globally inside the Activity's `dispatchKeyEvent()` callback via a high-level window router, rather than attaching isolated key listeners to layout adapters. Long presses are evaluated via `repeatCount` counts to prevent focus drift.
5.  **Provider Safety Controls**: The app shields users from panels blacklists by debouncing catch-up seeks by 1.5 seconds, filtering XC server credentials and tokens from crash logging frameworks, and ignoring D-pad key repeat clicks for stream reloads.

---

## 3. Guide / EPG Behavior

*   **Current-Time Anchoring**: Opening the guide automatically scrolls and shifts focus to the EPG cell representing the program currently broadcasting on the active channel.
*   **Future-Time Navigation**: Horizontal scrolling shifts the grid timeline into future cells. The current broadcast video continues playing in a background translucent layout.
*   **BACK Key Rules**:
    *   If the user has navigated horizontally into the future, clicking BACK snaps the grid viewport directly back to the current-time column.
    *   Clicking BACK a second time (or when already anchored to the current-time column) closes the guide overlay and restores fullscreen player focus.
*   **Channel / Category Looping**: Reaching the bottom channel cell loops focus back to the top channel card. Similarly, scrolling beyond the top category in the sidebar loops focus to the bottom category card.
*   **Focus Behavior**: Horizontal D-pad movements follow cell widths (program durations) dynamically. When scrolling channels vertically, the highlight maintains its horizontal timeline position as closely as possible.
*   **Catch-up Indicators**: A distinctive, pastel clock badge is drawn on past EPG program cells if the channel metadata indicates archive support, signalling playability.
*   **Guide Warmup & Performance**: At guide startup, visible-window EPG cells are loaded from the database into sorted in-memory lists (a `LinkedHashMap` keyed by Channel ID). Grid scrolling utilizes binary searches on these memory structures, achieving zero SQLite reads during scrolling.
*   **EPG Sync / Stall Handling**: Sync operations write to the DB in WAL (Write-Ahead Logging) mode. This allows the background EPG synchronizer to execute massive transactions without blocking the UI thread from querying active cells. A `PRAGMA busy_timeout = 3000` configuration handles temporary SQLite lockups.

---

## 4. Channel Bar / OSD Behavior

*   **Channel Preview**: When the horizontal OSD Channel Bar opens, it focuses the active channel card. Hovering over another channel card loads its current and next EPG metadata as a preview popup.
*   **Strip-Only Style**: The horizontal Channel Bar is rendered as a slim, compact bar of icons and titles at the bottom margin of the screen, leaving 85%+ of the screen space transparent so active video remains visible.
*   **Focus Behavior**: D-pad Left/Right scrolls between channel cards in the group. Clicking Up/Down returns focus to fullscreen video or opens the full guide.
*   **History Timeline**: Long-pressing D-pad Down opens a horizontal row of recently watched channels, allowing quick zapping back and forth.
*   **Performance / Caching**: Channel logo images are managed by Glide with a dedicated 250MB disk space cache, guaranteeing instant logo rendering during scrolls.
*   **OSD Timing & Overlays**: The OSD overlay (Info Bar, Channel Bar) fades out using decelerating transparency animations after a configurable timeout (typically 3–5s). Any remote button click resets the timer.

---

## 5. Catch-up / Timeshift Behavior

*   **Catch-up Metadata Ingestion**: Playlist parsers extract parameters from M3U `#EXTINF` tags: `catchup="default"`, `catchup-days="7"`, `catchup-source="URL"`. These are mapped to `playlists` and `channels` schema columns.
*   **Past Program Playability**: Past programs verify their archive availability boundaries:
    $$\text{Playable} = \text{Program.startTimeSeconds} > (\text{SystemTime} - \text{Playlist.catchupDays} \times 86400)$$
*   **Xtream Timeshift URLs**: Live stream paths are altered to timeshift structures:
    `.../timeshift/{username}/{password}/{duration_minutes}/{yyyy-MM-dd-HH-mm}/{stream_id}.ts`
*   **Flussonic Timeshift URLs**: Appends start and duration variables directly to file endpoints:
    `.../stream/mono-timeshift-{start_epoch}-{duration_seconds}.ts`
*   **Tokenized M3U Templates**: Resolves dynamic placeholder strings from the provider's `catchup-source` template, replacing `${start}`, `${end}`, `${duration}`, and `${offset}`.
*   **OSD Seek Overlay**: Tapping pause or seek controls opens a specialized progress timeline displaying program start/stop clocks, active program title, current stream offset, and real-time playback speed.
*   **Live Pause-to-Timeshift**: Pressing Pause on a live TV stream checks for archive support, generates the catch-up URL for the current epoch time, prepares the player with the timeshift source, and pauses it instantly.
*   **Hot-Swap Reloading**: Because HLS/TS archive streams do not support byte-range seeking, seeking computes the target epoch, re-generates the timeshift URL, and silent-reloads ExoPlayer via `player.prepare()`.
*   **Seek Debouncing**: Actual stream loading requests are throttled. Seeking inputs (slider drags, D-pad clicks) update the UI timeline instantly, but the player connection is delayed by **1.5 seconds** of inactivity to prevent connection spam.

---

## 6. Playlist / EPG Import Safety

*   **Streaming BufferedReader Parsing**: Playlist parsers process M3U streams line-by-line using a `BufferedReader` with an `InputStreamReader`. It avoids loading the full file text into RAM, safeguarding low-end TV boxes (1.5GB RAM) from Out-Of-Memory (OOM) crashes.
*   **Metadata Queue Buffering**: Metadata rows (`#EXTINF`) are pushed into an ordered token queue (`ArrayDeque`) to parse attributes sequentially without generating temporary large string allocations, reducing GC pressure.
*   **Incremental SQLite Diffing**: Imports execute inside a single `roomDatabase.withTransaction {}` block. It compares incoming groups and channels with the database:
    *   **INSERT**: New channels are added.
    *   **UPDATE**: Stream URLs or names are updated. User modifications (favorites, custom names) remain untouched.
    *   **SOFT DELETE**: Missing channels are marked with a `deleted_time` timestamp, keeping user customizations intact in case the provider restores the stream.
*   **Partial XMLTV Parsing**: The EPG parser first scans the XML `<channel>` elements and compares them with active channels in SQLite. It **only** parses `<programme>` tags for active, non-deleted channels, skipping 90%+ of redundant EPG parse allocations.
*   **Fault-Tolerant Load System**: If a playlist parses into 0 channels or fails with HTTP server errors (404, 503), the active database transaction is aborted, protecting the last known good state.
*   **SQLite Lock Prevention**: Enforces a 3,000ms database busy timeout. If locked, read requests wait for the write transaction to conclude instead of crashing.
*   **Blackout Cooldowns**: If a catastrophic database access failure occurs, the app registers a 1-hour blackout period. It blocks subsequent background update jobs during this hour to save CPU and battery.

---

## 7. Time / Offset / Calibration Rules

*   **`tvg_shift`**: The default timezone offset hour value specified in the playlist.
*   **Server Timezone**: Calibrates EPG start times based on server metadata values returned from Xtream Codes `/player_api.php` details.
*   **User Manual Offsets**: UI sliders allow category or channel-specific hour offsets (+/- 12 hours) to override server timezone parameters.
*   **System Clock Drift Calibration**: During startup, the network client executes a rapid HTTP `HEAD` call to the provider server. It compares the HTTP `Date` header to the local device time, computing a calibration delta ($\Delta_t$) applied to all subsequent timeshift epoch queries.

---

## 8. Provider Safety Rules

1.  **Seek Debouncing**: Apply a strict 1.5-second debounce window on timeshift seeks to avoid blacklisting of user IP/MAC credentials by provider panels.
2.  **Ignored Key Repeat events**: Exclude key events where `repeatCount > 0` on actions that trigger stream reloads (like channel zaps), protecting credentials from lockout boundaries.
3.  **One-Connection Limits**: If the playlist configuration dictates a single-stream limit, the player controller warns the user before launching a channel zap if a background recording task is active.
4.  **Credential Redaction**: Filter all user passwords, XC server authorization keys, Stalker MAC tokens, and signed playlist URLs from crash reports, analytics, and device logs.
5.  **Differentiated Cancellations**: Differentiate TCP socket drops (which trigger the exponential retry manager) from manual player releases (user zaps away), preventing false alarm reports.

---

## 9. PureFusionIPTV Gap Analysis

| Area | Behavior | Status | Recommended Action | Risk |
| ---- | -------- | ------ | ------------------ | ---- |
| **EPG / Guide** | Guide current-time anchoring | **Already implemented** | Recently improved; keep current focus layout | Low |
| **EPG / Guide** | Guide BACK behavior | **Already implemented** | Back snaps to current time, then exits; fixed | Low |
| **EPG / Guide** | Guide bottom-to-top channel loop | **Already implemented** | Keeps focus wrap-around active | Low |
| **EPG / Guide** | Category bottom-to-top loop | **Already implemented** | Loop focus wrap active | Low |
| **EPG / Guide** | Hide channel numbers option | **Already implemented** | Managed via user settings preferences | Low |
| **EPG / Guide** | Guide warmup / performance | **Partially implemented** | Implement EpgMemoryCache LinkedHashMap sorting | Medium |
| **EPG / Guide** | EPG parse-timeout hotfix | **Already implemented** | Fixed for Beta 9.22 candidate | Low |
| **EPG / Guide** | Optional Live TV clock | **Already implemented** | Clock widget layout overlay | Low |
| **EPG / Guide** | Theme/style presets | **Already implemented** | UI style presets; implemented | Low |
| **OSD / UI** | Channel Bar STRIP_ONLY preview | **Already implemented** | Compact horizontal list active | Low |
| **OSD / UI** | Manual EPG reliability | **Already implemented** | Improved; keep current mapping | Low |
| **Performance** | LoadControl buffer thresholds | **Missing** | Configure minBufferMs=1000, maxBufferMs=2500 in Player | Low |
| **Performance** | OkHttp socket pooling | **Missing** | Shared singleton OkHttpClient inside player factory | Low |
| **Performance** | 5-Second Surfing history delay | **Missing** | Coroutine job delay before writing history logs | Low |
| **Performance** | Buffering Watchdog System | **Missing** | Handler loop monitoring STATE_BUFFERING for 5s | Medium |
| **Timeshift** | Catch-up metadata ingestion | **Missing** | Schema database migration and M3U attributes parser | High |
| **Timeshift** | Polymorphic URL builder | **Missing** | Implement CatchupUrlBuilder for Xtream, Flussonic, M3U | High |
| **Timeshift** | Seek throttle debouncer | **Missing** | Implement 1.5s delay inside seek controller | Medium |
| **Security** | Log Credential Redaction | **Missing** | Clean logger inputs from user keys/tokens | Low |

---

## 10. Prioritized Roadmap

### Safe for Beta 9.22 (Low-Risk / High-Impact Stability)
*   **Custom Buffer Controls**: Configure ExoPlayer LoadControl bounds (minBufferMs=1000, maxBufferMs=2500, bufferForPlaybackMs=500, bufferForPlaybackAfterRebufferMs=1000) to instantly lower channel zapping times.
*   **Shared Connection Pool**: Bind all player DataSource loaders to a shared OkHttpClient singleton.
*   **Surfing History Filter**: Integrate the 5-second coroutine delay (`surfJob`) on database history logs to ignore surfing noise.
*   **Credential Redaction**: Filter passwords and tokens from logging utilities.

### Post-9.22 Polish (UX & Stability)
*   **Buffering Watchdog**: Integrate the 5-second `STATE_BUFFERING` watchdog to reconnect hung connections.
*   **Vertical LinearLayout OSD Container**: Align Info Bar and Channel Bar in a vertical stacking layout to automate overlay movements.
*   **Partial XMLTV parsing**: Parse only active, non-deleted channel EPG records to reduce CPU load.

### Larger Feature Phase (Catch-up / Timeshift)
*   **Timeshift DB Schema Migrations**: Add catch-up Type, Hours, and Source parameters to Room tables.
*   **Polymorphic URL Builder**: Implement the CatchupUrlBuilder parsing logic.
*   **Seek Debouncer & Hot-Swapping**: Implement the 1.5s seek debounce control and silent media source swapping.

---

## 11. Catch-up / Timeshift Implementation Blueprint

### I. Schema Additions (Room Entities)
Extend playlist and channel tables with catch-up variables:
```kotlin
// PlaylistEntity columns
@ColumnInfo(name = "catchup_type") val catchupType: Int = 0 // 0=None, 1=Xtream, 2=Flussonic, 3=Append
@ColumnInfo(name = "catchup_hours") val catchupHours: Int? = null
@ColumnInfo(name = "catchup_source") val catchupSource: String? = null

// ChannelEntity columns
@ColumnInfo(name = "catchup_hours") val catchupHours: Int? = null
@ColumnInfo(name = "user_catchup_time_offset") val userCatchupTimeOffset: Int? = null
```

### II. Metadata Import
*   **M3U Ingest**: Scan `#EXTINF` strings for `catchup="default"`, `catchup-days="7"`, and `catchup-source="URL"`.
*   **XC Ingest**: Store `catchup_type` and `catchup_hours` parsed from server stream responses.

### III. EPG Indicators
Evaluate cell playability inside the guide drawing logic:
```kotlin
fun canWatchCatchup(startTimeSeconds: Long, stopTimeSeconds: Long, catchupHours: Int): Boolean {
    val nowSeconds = System.currentTimeMillis() / 1000L
    val limitSeconds = nowSeconds - (catchupHours * 3600L)
    return stopTimeSeconds < nowSeconds && startTimeSeconds > limitSeconds
}
```
If true, draw a clock icon overlay on the program cell. Clicking initiates timeshift playback.

### IV. Polymorphic URL Builder
Format URL patterns based on provider parameters:
*   **Xtream Codes**: Injects `/timeshift/` and replaces path segments:
    `${scheme}://${host}:${port}/timeshift/${user}/${pass}/${duration_mins}/${yyyy-MM-dd-HH-mm}/${stream_id}.ts`
*   **Flussonic**: Appends epoch parameters:
    `${cleanUrl}-${start_epoch}-${duration_seconds}.ts`
*   **Append**: Replaces `${start}`, `${end}`, `${duration}`, and `${offset}` strings inside the playlist source template.

### V. Playback Handoff & Seek Debounce
*   **Hot-Swapping**: Seeks calculate the target timestamp, reconstruct the URL, and call `player.setMediaItem()` and `player.prepare()`.
*   **Seek Debounce**: Integrate a 1.5s timer. Seek actions update the UI timeline instantly, but dispatch the player reload command only after seek inputs cease for 1.5 seconds.
*   **Live-Pause**: Pause triggers on a live channel check archive support, query the epoch start time, prepare the timeshift stream, and pause it instantly.

---

## 12. Test Plan

### I. Unit Tests
*   `CatchupUrlBuilderTest`: Asserts path conversions for Xtream Codes, Flussonic, and M3U token placeholders.
*   `DebouncedSeekControllerTest`: Asserts that multiple seeks within 1.5s trigger only one playback reloading call.

### II. JVM-Safe Policy Tests
*   `LogRedactionTest`: Asserts that all passwords, MAC addresses, and token values are replaced with `[REDACTED]` markers in logs.

### III. Manual Android TV Checklist
- [ ] Zapping transitions complete in under 1.5s on Firestick devices.
- [ ] Leaving player buffering for 5s triggers a silent socket refresh.
- [ ] Channel Bar opens and slides the Info Bar upwards cleanly.
- [ ] Fast seeks do not trigger HTTP calls until remote clicks pause for 1.5s.
- [ ] Timestamps match local offset calendars without drifts.

### IV. Log Breadcrumbs
- [ ] `PlayPlayer: Resyncing hung stream socket after 5000ms buffering`
- [ ] `CatchupUrlBuilder: Resolved Xtream timeshift URL: http://.../timeshift/...`
- [ ] `LogFilter: Redacted credential parameters from output string`

---

## 13. Recommended Next Implementation Prompt

```text
PROMPT: Implement Phase 1 of the PureFusionIPTV Playback Roadmap.
Modify PlayerManager to integrate custom DefaultLoadControl buffers (minBufferMs=1000, maxBufferMs=2500, bufferForPlaybackMs=500, bufferForPlaybackAfterRebufferMs=1000). Setup a shared, singleton OkHttpClient DataSource factory to pre-warm TCP socket connection pools across zapping channels. Add a 5-second coroutine debouncer (surfJob) on playback history log database writes to screen out zapping noise.
```
