# PureFusionIPTV Playback & UX Optimization Roadmap — Clean-Room Architectural Analysis

This document outlines the **clean-room behavioral and architectural inventory, gap analysis, and developmental roadmap** for **PureFusionIPTV**. It is derived from a strict behavioral analysis of premium TV player paradigms (specifically TiViMate) to identify core mechanics, usability logic, edge-case mitigation, and provider safety rules.

---

## 1. EXECUTIVE SUMMARY

The goal of this analysis is to define a blueprint for **PureFusionIPTV** to achieve state-of-the-art IPTV playback stability, low-latency zapping, robust catch-up (timeshift) navigation, and smooth couch-focused TV layout transitions.

By adhering to a strict **clean-room methodology**, this document extracts **functional behaviors, user-experience flows, and architectural constraints** without referencing or copying any obfuscated decompiled code, assets, class hierarchies, or proprietary packages.

### Primary Usability Targets:
1. **Low-Latency Streaming**: Reduce zapping intervals from mobile-focused defaults (5–10s) to dedicated living-room bounds (<1.5s).
2. **Buffering Watchdog Resiliency**: Gracefully intercept and recover from frozen streams or hung sockets using a non-intrusive retry cycle.
3. **EPG-Integrated Timeshifting**: Translate timeline progress bar seeks on past programs into absolute epoch queries, resolving Xtream, Flussonic, and Append tokens dynamically.
4. **Couch-Optimized Navigation Stack**: Maintain deep focus sanity on remote controls, prevent layout overlaps, and preserve video background visibility using translucent HSL layers.
5. **Anti-Ban Provider Safety**: Ensure client queries remain throttled, preventing IPTV server IP blacklisting.

---

## 2. CLEAN-ROOM BEHAVIOR INVENTORY

The following inventory compiles useful high-end TV player behaviors that **PureFusionIPTV** should adopt, categorized by system area:

### I. Guide / EPG
* **Translucent Grid Overlays**: EPG grids render over active streams with translucent backgrounds. The video is dimmed but remains visible and playing.
* **Epoch-Bound Grid Queries**: Grid viewports query EPG databases using narrow Unix Epoch boundaries rather than wide queries, guaranteeing <10ms cursor response.
* **Time-Zone Offset Calibration**: Dynamic timezone alignment calculations that stack server time zone declarations, M3U shifts, and user offsets.
* **Catch-up Icon Flags**: Concluded EPG blocks display a visual "clock" or "watch" emblem if the stream provider database maps timeshift days > 0.
* **Anchor Snapping**: Opening the Guide aligns focus on the currently playing program cell, preventing scrolling misalignment.

### II. Channel Bar / Info Bar / OSD
* **Vertical Container Stacking**: The Info Bar (metadata) and Channel Bar (horizontal scroll) share a vertical linear container. Summoning the Channel Bar dynamically pushes the Info Bar upward to avoid overlap.
* **Zapping Info Display**: Changing channels displays a transient Info Bar showing current EPG program, next program, stream attributes (HD, SD, stereo tracks), and progress bounds.
* **5-Second Surfing Delay**: Do not immediately log zapped channels to history tables. Wait for a stable 5-second viewing window before updating history.

### III. Catch-up / Timeshift
* **Polymorphic URL Generation**: Resolves timeshift streams based on provider type (Xtream path modification, Flussonic filename mapping, Append M3U replacement tokens).
* **Epoch Seek Swapping**: Seeking on timeshift streams calculates absolute target epochs, re-resolves the streaming URL, and hot-swaps the running Player DataSource without restarting hardware decoders.
* **Live-to-Catchup Pausing**: Tapping Pause on live TV dynamically computes the active epoch offset, initializes the timeshift stream, and pauses it instantly.

### IV. Player Controls & Track Swapping
* **Seamless Audio/Subtitle Overrides**: Swap language tracks or subtitle formats by adjusting `TrackSelectionParameters` dynamically on the active player rather than re-prepare the media source.
* **Aspect Ratio Zoom Matrix**: Zoom, stretch, or fit layouts strictly at the View wrapper level (`SurfaceView.layoutParams`) instead of restarting hardware codecs.

### V. Settings / Customization
* **Slide-in leanback Drawers**: Preferences slide in from the screen edge as overlay panels over active video, keeping playback in view.
* **Remappable Keys**: Dispatch remote buttons contextually based on state (Player vs Guide vs Menu).

### VI. Performance / Caching / Warmup
* **Low-Latency LoadControl Boundaries**: Restrict buffer bounds: `minBufferMs = 1000`, `maxBufferMs = 2500`, `bufferForPlayback = 500`.
* **OkHttp Connection Pools**: Maintain warm TCP connection sockets across the same domain to bypass TLS renegotiation bottlenecks during zapping.
* **In-Memory EPG Decompression**: Stream `.xml.gz` downloads directly through a `GZIPInputStream` wrapper into memory parsers, eliminating physical write wear on internal flash.

### VII. Error Handling & Diagnostics
* **Stateful Buffering Watchdog**: If a stream enters `STATE_BUFFERING` and fails to recover or render its first frame within 5 seconds, declare a freeze, drop the hung socket, and recheck the source.
* **Exponential Backoff Retries**: If network disconnections occur, attempt re-prepares exactly 3 times with progressive delays (1s, 2s, 4s).
* **Background Auto-Recovery Loop**: If all retries fail, show a clean indicator card (`"An error occurred. Retrying..."`) and attempt reloads every 10s.

### VIII. Provider Safety
* **Seek Throttling**: Taps or slider drags update the UI timeline instantly but delay the actual HTTP stream request by 1.5 seconds.
* **Key Repeat Suppression**: Discard rapid successive keypresses on remappable player actions.

---

## 3. GAP ANALYSIS TABLE

This table maps **PureFusionIPTV**'s current status relative to these premium behavioral patterns:

| Behavior / Feature Area | Target Experience | PureFusionIPTV Status | Risk / Priority |
|---|---|---|---|
| **EPG Grid Rendering** | Direct Canvas pixel layouts, viewport epoch query bounds | Partially Implemented (Beta 9.21 Guide anchors fixed, parse hotfix done) | Medium Risk / Priority 1 |
| **Zapping Latency** | Low-latency buffering limits (`LoadControl` 1s/2.5s) | Missing (Uses Default ExoPlayer buffers) | Low Risk / Priority 1 |
| **Socket Reuse** | OkHttp Connection pool sharing across zaps | Missing (Uses default connection factory) | Low Risk / Priority 1 |
| **History surfing filter** | 5-second coroutine delay on writing to history database | Missing (Writes history instantly on zaps) | Low Risk / Priority 2 |
| **Layered Window Depth** | SurfaceView rendering behind translucent HSL overlay layers | Partially Implemented (Has style presets and clock overlays) | Low Risk / Priority 1 |
| **OSD Bar Collision** | Vertical LinearLayout auto-shifting stack for Info/Channel Bars | Missing (Info bar position is static) | Low Risk / Priority 2 |
| **Track Selection** | Override parameters on-the-fly without source reload | Partially Implemented (Basic track selection) | Medium Risk / Priority 2 |
| **Stream Freeze Recovery** | 5-second Buffering Watchdog to recheck dead sockets | Missing (Stalls on buffering indicator indefinitely) | Medium Risk / Priority 1 |
| **Dynamic Catch-up/Timeshift** | Interpolates Xtream, Flussonic, and M3U Append tokens | Missing (No catch-up playback support) | High Risk / Priority 3 (Feature Phase) |
| **Catch-up Seek Throttle** | 1.5-second debounce delay on timeshift requests | Missing | Medium Risk / Priority 3 |
| **Startup Date Sync** | Server HEAD Date offset check to adjust local clock drift | Missing | Medium Risk / Priority 2 |
| **In-Memory EPG Gzip** | Gzip decompression directly inside parsing stream | Missing (Writes decompressed payload to disk) | High Risk / Priority 2 |

---

## 4. PRIORITIZED ROADMAP

To transition **PureFusionIPTV** toward these targets, work is organized into progressive phases:

### Phase A: Safe for Beta 9.22 (Low-Risk / High-Impact Performance)
1. **ExoPlayer LoadControl Tuning**: Re-configure buffer thresholds inside `PlayerManager` (1s min, 2.5s max, 500ms playback startup) to instantly drop zapping latency.
2. **Shared OkHttp Client**: Build a singleton `OkHttpClient` connection factory for `OkHttpDataSource` to reuse sockets.
3. **Playback History Surfing Filter**: Integrate a 5-second coroutine debouncer on history log entries.

### Phase B: Post-9.22 Polish (UX & Stability)
1. **Buffering Watchdog System**: Implement a 5-second timeout listener on `STATE_BUFFERING`. Re-initialize the stream source silently if triggered.
2. **Vertical LinearLayout OSD Container**: Structure Info Bar and Channel Bar views inside a vertical container to shift layouts when the channel list opens.
3. **In-Memory EPG Decompression**: Stream XMLTV files straight from `GZIPInputStream` into `XmlPullParser`, avoiding flash wear.

### Phase C: Future Feature Phases (Catch-up / Timeshift)
1. **Dynamic URL Catch-up Resolvers**: Construct timeshift URLs for Xtream, Flussonic, and Append schemas.
2. **Epoch Seek Swapping**: Catch seek events, calculate the target Epoch timestamp, and swap player sources.
3. **Time offset drift calibration**: Sync local TV clocks with server headers.

---

## 5. CATCH-UP / TIMESHIFT BLUEPRINT

This clean-room plan defines how to implement Catch-up (Timeshift) features safely within PureFusionIPTV:

```
                  [ EPG Click Concluded Program ]
                                 │
                                 ▼
         [ Query database: getEffectiveCatchup(channel) ]
                                 │
                 ┌───────────────┴───────────────┐
                 ▼ (Hours == null or <= 0)       ▼ (Hours > 0 / Valid)
            [ Return: Not Playable ]      [ Resolve catchup_type & source ]
                                                 │
                                                 ▼
                                     [ Format Target URL ]
                                  (Xtream / Flussonic / Append)
                                                 │
                                                 ▼
                                     [ Load Stream Source ]
```

### I. Metadata Ingestion
* **M3U Headers**: During ingestion parsing, check `#EXTINF` strings for:
  * `catchup="default"` or `catchup="shift"`
  * `catchup-days="X"` or `catchup-hours="Y"`
  * `catchup-source="URL_TEMPLATE"`
* **Database Migration**: Add `catchup_type`, `catchup_hours`, and `catchup_source` columns (nullable) to `playlists` (global configurations) and `channels` (specific overrides).

### II. EPG Selection Logic
During EPG drawing, determine playability:
```kotlin
fun isProgramPlayable(program: Program, catchupHours: Int): Boolean {
    val currentSec = System.currentTimeMillis() / 1000
    val startSec = program.startTimeEpochSeconds
    val limitSec = currentSec - (catchupHours * 3600)
    
    return startSec < currentSec && startSec > limitSec
}
```
If playable, draw a small translucent clock indicator (`@drawable/ic_catchup_clock`) on the top-right edge of the EPG program block.

### III. URL Generation Engine
Format URL parameters according to provider type:
* **Xtream Codes**: Path transformation:
  $$\text{Url} = \text{server} + \text{"/timeshift/"} + \text{user} + \text{"/"} + \text{pass} + \text{"/"} + \text{durationMins} + \text{"/"} + \text{formattedTime} + \text{"/"} + \text{streamId}$$
* **Flussonic**: Filename appendage:
  $$\text{Url} = \text{basePath} + \text{"-"} + \text{startEpoch} + \text{"-"} + \text{durationSecs} + \text{".ts"}$$
* **Append**: String token replacements on template variables (`${start}`, `${duration}`).

### IV. Seek Strategy
* **Debounced seeking**: Do not launch network request threads on every drag update. Trigger progress updates locally on screen. Dispatch the socket reload **1.5 seconds** after seek input stops.
* **Epoch-Based hot-swapping**: Since timeshift streams are HLS media feeds, seek actions construct a fresh URL with the target epoch timestamp and call `player.prepare()`.

---

## 6. PROVIDER SAFETY REVIEW

To safeguard accounts and ensure compliance with IPTV panel security rules:

1. **Debounce Catch-up Seek Requests**: Restrict active connections. Maintain a minimum of **1.5 seconds** between consecutive stream requests.
2. **Suppress Key Repeat Actions**: Track D-pad long-presses. Do not trigger socket reloads while buttons are held down. Only execute the action on key-up.
3. **Avoid Credential Logging**: Redact sensitive parameter keys (`username`, `password`, `token`, MAC addresses) from log diagnostics and debug exports.
4. **Prevent Connection Limit Exhaustion**: On playlists with strict single-connection limits, warn the user before tuning away if an active background PVR task is recording.

---

## 7. UX PARITY REVIEW

To replicate premium TV usability flows:

* **EPG Focus Anchor**: Opening the TV Guide should snap focus to the cell representing the active channel's current EPG program.
* **BACK Key Lifecycle**:
  * If the sidebar menu or settings is open: **Close drawer**.
  * If the TV Guide is open over video: **Close TV Guide** (return focus to player).
  * If the video is playing in fullscreen: **Summon TV Guide** (overlay player).
  * If the EPG is scrolled into the future: Clicking BACK should **snap focus back to current time**, and a second click closes the Guide.
* **OSD Panel Depth**: Video plays behind a static translucent scrim to protect text legibility, avoiding CPU-heavy Gaussian blurs.
* **List Looping**: Categories and channel lists should cycle (pressing down on the last item loops focus to the top item).

---

## 8. TESTING PLAN

To ensure system stability, implement this automated and manual testing plan:

### I. Unit Tests
* **`CatchupUrlBuilderTest`**: Test formatting functions against Xtream, Flussonic, and M3U URLs. Validate parameter extraction and timezone shifts.
* **`DebouncedSeekControllerTest`**: Assert that sequential seek requests within 1.5 seconds get throttled, executing only the final request.

### II. Integration Tests
* **`DatabaseCascadeQueryTest`**: Verify that EPG joined queries fallback correctly from Channel Overrides -> Category Overrides -> Playlist Configs -> Global Defaults.

### III. Manual TV-Stick Checklist
- [ ] Zapping latency is under 1.5 seconds.
- [ ] Tapping D-pad Center shifts Info Bar upward and displays Channel Bar at bottom.
- [ ] Seeking on Catch-up plays from the target playhead.
- [ ] Rapid seeking doesn't crash the player or throw 403 authorization failures.
- [ ] Android TV remote long-press on Center button opens Quick Action panel.

---

## 9. RECOMMENDED NEXT IMPLEMENTATION PROMPT

Use the following prompt when you are ready to implement these stability changes:

```text
PROMPT: Implement Phase A of the PureFusionIPTV Playback Roadmap.
Modify PlayerManager to integrate custom DefaultLoadControl buffers (minBufferMs=1000, maxBufferMs=2500, bufferForPlaybackMs=500). Setup a shared, singleton OkHttpClient DataSource factory to pre-warm TCP socket connection pools across zapping channels. Add a 5-second coroutine debouncer (surfJob) on playback history log database writes to screen out zapping noise.
```
