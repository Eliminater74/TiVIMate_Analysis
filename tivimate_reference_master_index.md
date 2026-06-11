# TiViMate Decompiled Source Reference — Master Index File

This master index maps the complete technical audit and architectural blueprints extracted from the decompiled **TiViMate** source code. It organizes the files by feature area, summarizes key findings, specifies their relevance to **PureFusionIPTV**, and designates developmental priority, risk level, and suggested Beta 10 implementation phases.

---

## PRIORITY DEFINITIONS
*   **P0**: Critical core logic / Must preserve (Do not regress).
*   **P1**: Beta 10 Core Improvement / Essential for TV performance.
*   **P2**: Beta 10 Performance & UX Polish / Elite television aesthetics.
*   **P3**: Future Architecture / Advanced features.
*   **P4**: Experimental / Optional later phase.

---

## 1. TECHNICAL AUDIT REPORTS INDEX

| File Name | Feature Area | Key Findings | PureFusion Relevance | Priority | Risk | Beta 10 Phase |
|---|---|---|---|---|---|---|
| [tivimate_guide_performance_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_guide_performance_analysis.md) | **EPG Grid Rendering** | Custom Canvas direct pixel layouts, binary search viewport queries, 90-min RAM caches | Bypass standard RecyclerView stutters and lag completely | **P0** | High | Phase 1 |
| [tivimate_startup_vod_ui_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_startup_vod_ui_analysis.md) | **Startup & Search** | SQLite WAL pre-warming, last channel prioritize startup, FTS4 matching, glassmorphic timers | Secure under-300ms zapping Cold Start times | **P0** | Medium | Phase 1 |
| [tivimate_cold_start_engine_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_cold_start_engine_analysis.md) | **Cold Start Sequence** | Asynchronous DB pre-warming, Fast preferences last played channel, deferred background syncs | Achieve sub-300ms boot playback speeds | **P0** | Medium | Phase 1 |
| [tivimate_network_player_tuning_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_network_player_tuning_analysis.md) | **Player & Socket Tuning** | OkHttp connection pooling, 1-sec LoadControl bounds, direct Gzip EPG decompress stream | Achieve sub-1-second channel changes without storage disk wear | **P0** | Medium | Phase 1 |
| [tivimate_search_architecture_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_search_architecture_analysis.md) | **High-Speed Search** | SQLite FTS4 virtual tables, MATCH operators, narrow query widths, debounced input flows | Instant search-as-you-type without typing freezes or ANRs | **P0** | Low | Phase 1 |
| [tivimate_playlist_parsing_stalker_xc.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_parsing_stalker_xc.md) | **Playlist Parsing** | `#EXTINF` tokenizers, Xtream REST APIs, Stalker MAC handshakes and silent token refresh loops | Implement secure MAG box and XC integrations on standard layouts | **P1** | High | Phase 1 |
| [tivimate_playlist_update_safety_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playlist_update_safety_analysis.md) | **Sync Safety** | Incremental update transactions, soft deletes, user override preservation filters | Prevent playlist wipes or data loss on server-side failures | **P1** | Medium | Phase 1 |
| [tivimate_dvr_favorites_history_audit.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_dvr_favorites_history_audit.md) | **DVR, Favorites & History** | Exact Alarm clock wakes, partial WakeLocks, direct TS piping, 5-sec history surfing delays | Schedule reliable background recordings and surf filters | **P1** | High | Phase 2 |
| [tivimate_licensing_device_management.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_licensing_device_management.md) | **Licensing & Licensing Checks** | SHA-256 TV fingerprints, JWT local tokens, server-side device counts, 8-device scalability | Build premium licensing validation structures securely | **P1** | High | Phase 2 |
| [tivimate_channel_group_customization_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_channel_group_customization_analysis.md) | **Group Customization** | Junction mapping tables (`group_channel_mappings`), custom renaming, atomic index shifts | Enable users to reorder and copy/move channels between groups | **P2** | Medium | Phase 2 |
| [tivimate_playback_controls_settings_audit.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_playback_controls_settings_audit.md) | **Playback & Settings UI** | SurfaceView scaling zoom matrices, Media3 track overrides, Audio sink sync, slide preferences | Seamless aspect ratio scales and side sliding Leanback drawers | **P2** | Low | Phase 2 |
| [tivimate_parental_controls_security_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_parental_controls_security_analysis.md) | **Parental Locks** | SHA-256 PIN hashing, separate hidden vs locked database states, temporary unlocked sessions | Restrict adult categories securely via PIN cards | **P2** | Low | Phase 2 |
| [tivimate_vod_metadata_resume_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_vod_metadata_resume_analysis.md) | **VOD & Playback Resume** | Series/Seasons/Episodes schemas, 10s playhead resume markers, Glide image cache scale | Resume movie playback exactly where the user stopped | **P2** | Medium | Phase 2 |
| [tivimate_transparency_video_overlay_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_transparency_video_overlay_analysis.md) | **Layered Transparency** | Layered window stack, SurfaceView z-indexing, legibility scrim views, translucent styles XML | Display active video behind EPG and search overlays without lag | **P2** | Low | Phase 2 |
| [tivimate_multiview_architecture_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_multiview_architecture_analysis.md) | **Multiview Engine** | Concurrent ExoPlayer instances, dynamic audio focus, hardware decoder limits, software fallbacks | Watch up to 9 channels concurrently with active D-pad audio focus | **P2** | High | Phase 3 |
| [tivimate_auxiliary_systems_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_auxiliary_systems_analysis.md) | **Auxiliary Systems** | Fuzzy string logo normalizers, startActivityForResult external progress syncs, sleep timers | Re-sync movie resume states from VLC and fuzzy match channel icons | **P2** | Low | Phase 2 |
| [tivimate_error_handling_low_end_device_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_error_handling_low_end_device_analysis.md) | **Device Optimization** | ActivityManager memory scaling, Software decoders (FFmpeg), VLC intents, exponential backoff | Ensure steady 60fps speeds on cheap 1.5GB RAM TV sticks | **P2** | Medium | Phase 3 |
| [tivimate_theme_effects_engine.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_theme_effects_engine.md) | **Theme & Visual Engine** | ContextThemeWrapper dynamic styles, glassmorphic stroke styles, focused card elevations | Swap visual color themes on-the-fly without ExoPlayer reload gaps | **P3** | Low | Phase 3 |
| [tivimate_backup_restore_deep_dive.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_backup_restore_deep_dive.md) | **Backup & Restore** | PRAGMA checkpoints, ZIP/GZIP streams, SAF file selectors, database closing hot-swaps | Package settings and databases securely with rollback safeguards | **P1** | High | Phase 2 |
| [tivimate_usability_insights_guide.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_usability_insights_guide.md) | **Architectural Rules** | Master list: what developers can replicate (SQL, HSL) vs what they must avoid (R8 Java code) | Strict developer safeguard to avoid copying decompiled artifacts | **P0** | Low | Phase 1 |
| [tivimate_master_reference_catalog.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_master_reference_catalog.md) | **Reference Catalog** | Comprehensive index of all 22 files, detailing exactly what is explained in every section of each file | Ultimate index to easily locate visual overlays, database queries, and settings | **P0** | Low | Phase 1 |

---

## PureFusionIPTV Beta 10 Development Roadmap Summary

To bring **PureFusionIPTV** to a world-class level, structure your active development in these consecutive phases:

### Phase 1: Core Performance & Ingestion (P0-P1)
1.  **Configure SQLite WAL Mode**: Warm the database and implement composite timeline index mappings (`idx_tvg_programs_channel_time`).
2.  **OkHttp DataSource Pre-warming**: Implement Media3 OkHttp extension datasource factories with socket pools to reduce zapping times.
3.  **Direct Canvas Guide Drawing**: Build custom graphics rendering paths to bypass vertical-horizontal RecyclerView stutters entirely.
4.  **High-Speed FTS4 Search**: Mirror search strings into FTS4 virtual tables, querying via MATCH operators and a 300ms input debounce Flow.
5.  **Streaming Gzip Parsers**: Pipe `.xml.gz` downloads directly through `GZIPInputStream` to pull XML tokens in memory without disk wear.

### Phase 2: User UX, Customization & PVR (P1-P2)
1.  **Surfing Delay History**: Implement coroutine-based debounced filters (`surfJob`) to screen out zapping noise from history.
2.  **Secure Parental PINs**: Apply SHA-256 PIN hashing in private storage and distinct database visibility locks.
3.  **Exact DVR Alarms & WakeLocks**: Coordinate AlarmManager wake broadcasts with Foreground PVR services holding partial WakeLocks.
4.  **Cascading Settings Resolution**: Implement falling-back preferences (Channel -> Category -> Playlist -> Global) using nullable database fields.
5.  **Layered Translucent Layouts**: Build transparent systems XML styles and SurfaceView layouts to permit video background bleeding.

### Phase 3: Hardware Resiliency & Aesthetics (P2-P3)
1.  **Memory-Aware Scaling**: Detect low system RAM via ActivityManager and drop Glide caches and EPG memory buffers accordingly.
2.  **Software Decoder Fallbacks**: Add FFmpeg code fallbacks to handle `DECODER_INIT_FAILED` crashes gracefully on low-end chips.
3.  **Glassmorphic Overlay Themes**: Implement ContextThemeWrapper dynamic overlays for smooth visual shifts without stream reloading.
4.  **Dynamic Scrim Animators**: Build overlay scrim views that animate alpha from 0.0f to 0.6f on menu clicks to protect legibility.
5.  **Multi-View Audio Focus**: Build multi-player grids that toggle volume (1.0f vs 0.0f) dynamically based on active D-pad cursor focus.
