# TiViMate Decompiled Source Reference — Master Reference Catalog

This master catalog serves as the absolute index for the complete technical reference library extracted from the decompiled **TiViMate** source code. It categorizes all **22 architectural analyses, guidelines, and reference reports** created for you, detailing exactly what is explained in **every section of each file** to ensure quick recall and implementation mapping.

---

## CATEGORY 1: CORE PERFORMANCE & GRID SYSTEMS

### 1. EPG Grid Performance Analysis
*   **File Path**: [tivimate_guide_performance_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_guide_performance_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: The 3 Pillars of EPG Fluidity**: Dissects how TiViMate eliminates scrolling lag by loading EPG data once into sorted RAM arrays, using custom direct-to-Canvas pixel drawing, and deploying a 90-minute RAM TTL cache.
    *   **Section 2: Two-Table Database Schema**: Breaks down why EPG data is separated into internally managed (`my_programs`) and externally mapped (`tvg_programs`) SQLite tables, using Unix Epoch Seconds (not milliseconds) to halve database index page width. Includes the optimized EPG join query.
    *   **Section 3: Three-Layer Cache Architecture**: Explores the 90-minute RAM memory cache, the 250MB Glide logo disk cache, and the 3-second database busy timeout locks.
    *   **Section 4: Custom Canvas Renderer vs RecyclerView**: Outlines why standard RecyclerViews fail on horizontal/vertical TV grids, demonstrating the math behind direct Canvas pixel coordinates ($startX$ and $width$).
    *   **Section 5: EPG Time Zone Offset Stacking**: Explains the triple timezone offset corrections (`tvg_shift` + `server_timezone` + user offsets).
    *   **Section 6: PureFusionIPTV Implementation Blueprints**: Provides complete Kotlin templates for the `EpgMemoryCache` singleton and custom `EpgCanvasView` rendering framework.

### 2. Cold Start Engine & Boot Sequence
*   **File Path**: [tivimate_cold_start_engine_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_cold_start_engine_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Cold Start Flow**: Outlines sub-300ms boot behaviors, instant stream playback recovery, and asynchronous UI populations.
    *   **Section 2: Chronological Boot Sequence Stages**: Dissects the chronological lifecycle steps (asynchronous Room database pre-warming, Fast SharedPreferences lookups, immediate player buffering, lazy EPG grid Canvas line binding, and background sync deferrals).
    *   **Section 3: Boot Speed Secrets Brief**: Compares the standard Android TV boot bottlenecks to TiViMate's lightweight design solutions.
    *   **Section 4: Beta 10 Actionable Roadmaps**: Actionable low, medium, and high-risk performance recommendations.

### 3. High-Speed Search Architecture
*   **File Path**: [tivimate_search_architecture_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_search_architecture_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Search-as-you-Type Flow**: Outlines segmented results (Live, Movies, TV Shows) and instant remote key transitions.
    *   **Section 2: SQLite FTS4 Virtual Tables**: Compares FTS MATCH operators to slow `LIKE` queries, demonstrating why FTS B-trees return queries under 1 millisecond.
    *   **Section 3: Key Queries Observed**: Dissects the compiled queries from `C3214.java` (Case 18) and `C3220.java` (Case 11), demonstrating EPG past boundaries and image-width optimization tricks.
    *   **Section 4: Conceptual Kotlin Blueprints**: Offers complete code guidelines for Room `@Fts4` entities, MATCH DAOs, and a 300ms input debounced flow.
    *   **Section 5: TV Focus & Keyboard Lock**: Explains how keyboard cursor grids lock focus, avoiding automatic jumps when results refresh in the background.

---

## CATEGORY 2: PLAYLIST INGESTION & NETWORK PLAYER TUNING

### 3. Playlist Ingestion, M3U & API Clients
*   **File Path**: [tivimate_playlist_parsing_stalker_xc.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_parsing_stalker_xc.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: M3U Ingestion Attributes**: Dissects `#EXTINF` metadata tokens (`tvg-id`, `tvg-logo`, `group-title`, `catchup-days`, `catchup-source`) and how they parse sequentially into category rows.
    *   **Section 2: Xtream Codes (XC) API Sequence**: Explores the 4-step sync path (Authentication `/player_api.php`, dynamic categories, streams mapping, and short EPG queries).
    *   **Section 3: Stalker Portal MAG Box Emulation**: Breaks down the Stalker handshake sequence (`mac=00:1A:79`), channel indices, and dynamic link generation (`action=create_link`). Includes the background token quiet-refresh watchdog.

### 4. Playlist Update Safety & State Preservation
*   **File Path**: [tivimate_playlist_update_safety_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_update_safety_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Incremental Diffing Transactions**: Details why TiViMate never wipes tables during refreshes, using comparative updates (INSERT, UPDATE, SOFT DELETE) wrapped in `database.withTransaction {}`.
    *   **Section 2: Preserving User State**: Explores how user modifications (custom names, EPG bindings, and favorites) are protected during imports by storing overrides in distinct columns.
    *   **Section 3: Fault-Tolerant Sync Safeties**: Details response verification boundaries and silent background failures that prevent playlist corruption when servers return HTTP 503 or 404.

### 5. Network Stream Tuning & ExoPlayer LoadControl
*   **File Path**: [tivimate_network_player_tuning_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_network_player_tuning_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Custom ExoPlayer LoadControl Settings**: Breaks down compiled boundaries in `p392.C4655` (`minBufferMs=1000`, `maxBufferMs=2500`, `bufferForPlayback=500`), achieving sub-1-second channel change times.
    *   **Section 2: Media3 OkHttp DataSource Integration**: Details how TCP connection pooling and HTTP/2 multiplexing bypass TLS handshakes during active zapping.
    *   **Section 3: Blueprints**: Provides clean Kotlin implementation guidelines for OkHttpClient connection pools, custom DefaultLoadControl buffers, and in-memory Gzip streaming EPG decompression wrappers.

---

## CATEGORY 3: DVR, FAVORITES & PLAYBACK HISTORY

### 6. PVR & Scheduled Background Recordings
*   **File Path**: [tivimate_pvr_recording_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_pvr_recording_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: DVR Cold-Start Wake**: Explains using Android `AlarmManager` exact clocks (`setExactAndAllowWhileIdle`) and persistent Foreground Services (`PvrService`) to wake the CPU and survive TV CEC standby.
    *   **Section 2: Direct TS Piping**: Demonstrates direct file dumping from network sockets to disk using a `BufferedOutputStream`, maintaining under 2% CPU usage.
    *   **Section 3: SQLite Query Bindings**: Shows decompiled `recordings` queries mapping scheduled items (`RecordingForPush`) and dynamically updating target file paths (`C3220.java`).
    *   **Section 4: Kotlin Blueprints**: Complete Room entity schemas, direct streaming pipe helper classes, and foreground PVR services holding partial WakeLocks.

### 7. DVR WakeLocks, Favorites & Surfing History
*   **File Path**: [tivimate_dvr_favorites_history_audit.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_dvr_favorites_history_audit.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Background PVR Wake**: Deeper dive into CPU WakeLock lifecycle flags during active recordings.
    *   **Section 2: Double-Buffered Favorites**: Resolves favorites dynamically across multiple playlists in real-time using `is_favorite` SQLite flags.
    *   **Section 3: The 5-Second Surfing Delay Filter**: Explains how a coroutine-based delay (`surfJob`) filters quick-zapping noise from history, updating `last_turn_on_time` and EPG snapshots on-the-fly only when users stay on a channel.

### 8. Catch-up Timeshift & Reminders Engine
*   **File Path**: [tivimate_catchup_recording_reminders_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_catchup_recording_reminders_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: EPG-Integrated Catch-up**: Explains dynamic time interpolation for Xtream, Flussonic, and Append timeshift templates, mapping seeks to absolute Epoch seconds.
    *   **Section 2: DVR Conflict Management**: Details multi-stream recording limits and error handling when account stream connection limits are hit.
    *   **Section 3: EPG Reminders & Alarms**: Shows how the system schedules program alarms, pops transparent dialog overlays, and warns before zapping to protect active PVR streams on single-stream accounts.

---

## CATEGORY 4: CUSTOMISATION & LIVING ROOM UX

### 9. Aspect Ratio Zooming, Audio Sync & Settings Drawer
*   **File Path**: [tivimate_playback_controls_settings_audit.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playback_controls_settings_audit.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Aspect Ratio scaling**: Shows how Normal, Fit, Fill, Stretch, and Zoom are handled at the View layout layer (`SurfaceView.layoutParams`) without codec restarts.
    *   **Section 2: Media3 Multi-Track overrides**: Swapping audio languages and subtitles dynamically via `TrackSelectionParameters`.
    *   **Section 3: Audio sync offset**: Adjusting playhead buffers in `DefaultAudioSink` to correct timestamp drift (+/- 2000ms).
    *   **Section 4: Side sliding Leanback preference drawers**: Creating vertical-aligned settings panels that animate smoothly (`translationX`) over active transparent video.

### 10. Remote Input & Key Mapping System
*   **File Path**: [tivimate_remote_input_menu_architecture.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_remote_input_menu_architecture.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Window Key Interception**: Audits how D-pad buttons are captured globally at `MainActivity.dispatchKeyEvent` and mapped to context-sensitive actions, handling long presses via repeat counts.
    *   **Section 2: Quick Menu Customization**: Dynamically inflates custom horizontal actions grids based on serialized preference arrays.

### 11. Parental Control Locks & PIN Hashing
*   **File Path**: [tivimate_parental_controls_security_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_parental_controls_security_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Cryptographic PIN Security**: Storing parental lock PINs inside private storage as **SHA-256 hashed strings** instead of plain text, hashing user inputs before comparison.
    *   **Section 2: Hidden vs Locked Items**: Defines hidden groups/channels (completely filtered from guide queries: `is_visible == 0`) and locked elements (padlock overlays requiring PIN).
    *   **Section 3: Setting & Backup locks**: Protecting preference folders and export profiles dynamically.

---

## CATEGORY 5: THEME ENGINE & VISUAL EFFECTS

### 12. Dynamic Theme Engine overlays
*   **File Path**: [tivimate_theme_effects_engine.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_theme_effects_engine.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: ContextThemeWrapper Swapping**: Details how `WindowCallbackC4401.java` resolves `R.attr.u_res_0x7f04000b` and applies overlay style styles on-the-fly (`applyStyle()`), changing accents (Blue, Teal, Indigo) without activity recreation or stream stutters.
    *   **Section 2: Visual Effects Specifications**: Outlines glassmorphic stroke border styles, dynamic focus border state selectors (`accent_focus_glow`), 3D shadow elevations (`translationZ`), and alpha cross-fade animation curves.

### 13. Layered Window Transparency & Background Video
*   **File Path**: [tivimate_transparency_video_overlay_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_transparency_video_overlay_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Layered Window Stack**: Coordinates the three-layer layout (Z-index: 0: SurfaceView, Z-index 1: Scrim View, Z-index 2: UI Overlays).
    *   **Section 2: System Translucent Theme**: Manifest and style XML parameters (`android:windowBackground = @android:color/transparent` and `android:windowIsTranslucent = true`) that permit background video bleeding.
    *   **Section 3: GPU Conservation**: Bypasses expensive real-time Gaussian blurs on budget TV sticks, utilizing static alpha HSL colors and gradient scrims.

---

## CATEGORY 6: SETTINGS, BACKUPS & HARDWARE OPTIMIZATIONS

### 14. Backup & Restore Deep-Dive
*   **File Path**: [tivimate_backup_restore_deep_dive.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_backup_restore_deep_dive.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Backup Packaging Pipeline**: Forcing database checkpoints (`PRAGMA wal_checkpoint(FULL);`) to flush WAL writes, compressing SQLite `.db` and preferences XML files inside a `.tmb` zip, and writing metadata checksums.
    *   **Section 2: Safe Restore Swaps**: Reading version code guards to block schema downgrades, closing active database connections, backing up rollback `.bak` copies, and scheduling `AlarmManager` self-restarts before killing the process to refresh cached SharedPreferences.

### 15. Low-End Device Optimizations & Software Fallback
*   **File Path**: [tivimate_error_handling_low_end_device_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_error_handling_low_end_device_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Memory Class Profiling**: Querying `getMemoryClass()` to down-scale Glide image bounds and EPG RAM cache sizes on low-memory TV sticks.
    *   **Section 2: Software Decoder Fallbacks**: Switching to FFmpeg software decoders on `DECODER_INIT_FAILED` hardware limit exceptions.
    *   **Section 3: Intent-Based VLC/MX Player Integration**: Dispatches implicit `ACTION_VIEW` intents with injected HTTP headers and User-Agents to launch external players.

### 16. Licensing, JWT & 8-Device Limits
*   **File Path**: [tivimate_licensing_device_management.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_licensing_device_management.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Device Fingerprinting**: Generating SHA-256 device IDs using ANDROID_ID, Build manufacturer, model, and product parameters.
    *   **Section 2: Licensing Handshake**: Verifying user-credentials, checking device registration count, and return signed JWT tokens stored locally in SharedPreferences for offline validity.
    *   **Section 3: 8-Device Limit Scalability**: Structural advice on central databases to support your 8-device limits.

### 17. Auxiliary Systems (Logo Matching, External Resume & Sleep Timers)
*   **File Path**: [tivimate_auxiliary_systems_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_auxiliary_systems_analysis.md)
*   **Detailed Section Breakdown**:
    *   **Section 1: Inexact Fuzzy Logo Matcher**: Normalizing strings to lowercase and stripping punctuation/whitespaces to match local directories.
    *   **Section 2: External Playback Progress Sync**: Launching third-party apps via `startActivityForResult` and extracting current playhead markers from VLC and MX Player bundles in `onActivityResult` upon close.
    *   **Section 3: Standby Sleep Timers**: Implementing background timers, warnings dialog overlays, and CEC standby finish commands.

---

## CATEGORY 7: ARCHITECTURAL GUIDES & INDEXES

### 18. Master Reference Index
*   **File Path**: [tivimate_reference_master_index.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_reference_master_index.md)
*   **Detailed Section Breakdown**:
    *   A central matrix listing all analysis reports by feature area, key findings, priority (P0 to P4), risk, and suggested Beta 10 implementation phase.

### 19. Architectural Usability Insights Guide
*   **File Path**: [tivability_insights_guide.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_usability_insights_guide.md)
*   **Detailed Section Breakdown**:
    *   A developer safeguard specifying exactly what patterns you can adopt (SQL string queries, HSL visual colors, buffered tokenization) vs what you must avoid (obfuscated Java classes, R8 layout measurements).
