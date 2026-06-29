<p align="center">
  <img src="https://img.shields.io/badge/TiViMate-v5.2.0-blueviolet?style=for-the-badge&logo=android&logoColor=white" alt="TiViMate Version"/>
  <img src="https://img.shields.io/badge/Analysis-Deep_Dive-critical?style=for-the-badge&logo=bookstack&logoColor=white" alt="Analysis Type"/>
  <img src="https://img.shields.io/badge/Platform-Android_TV-green?style=for-the-badge&logo=androidtv&logoColor=white" alt="Platform"/>
</p>

<p align="center">
  <img src="https://komarev.com/ghpvc/?username=Eliminater74&repo=TiVIMate_Analysis&label=Repo+Views&color=brightgreen&style=flat-square" alt="Repo Views"/>
  <img src="https://img.shields.io/github/stars/Eliminater74/TiVIMate_Analysis?style=flat-square&color=yellow&logo=github" alt="Stars"/>
  <img src="https://img.shields.io/github/forks/Eliminater74/TiVIMate_Analysis?style=flat-square&color=blue&logo=github" alt="Forks"/>
  <img src="https://img.shields.io/github/watchers/Eliminater74/TiVIMate_Analysis?style=flat-square&color=orange&logo=github" alt="Watchers"/>
  <img src="https://img.shields.io/github/last-commit/Eliminater74/TiVIMate_Analysis?style=flat-square&color=purple" alt="Last Commit"/>
  <img src="https://img.shields.io/github/repo-size/Eliminater74/TiVIMate_Analysis?style=flat-square&color=red" alt="Repo Size"/>
  <img src="https://img.shields.io/github/license/Eliminater74/TiVIMate_Analysis?style=flat-square" alt="License"/>
</p>

---

# 📺 TiViMate v5.2.0 — Complete Decompiled Source Analysis

> **The most comprehensive architectural deep-dive ever created for TiViMate IPTV Player.** Over **50 technical analysis documents** dissecting every system — from the sub-300ms cold start engine, to the EPG Canvas renderer, Stalker Portal emulation, PVR recording pipeline, and everything in between.

---

## 📊 Repository Statistics

| Metric | Count |
| :--- | :---: |
| 📝 Analysis Documents (`.md`) | **52** |
| ☕ Decompiled Java Source Files | **6,416** |
| 📦 Source Packages | **783** |
| 🎨 Resource Files (XML, drawables, etc.) | **1,587** |
| 📁 Resource Directories | **143** |
| 📐 Total Analysis Size | **~594 KB** |
| 🎯 Target APK Version | **5.2.0 (build 5208)** |
| 📱 Min SDK | **23 (Android 6.0)** |
| 🎯 Target SDK | **34 (Android 14)** |

---

## 🗂️ Repository Structure

```
TiVIMate_Analysis/
├── 📝 *.md                    # 52 architectural analysis documents (this folder)
├── 📁 sources/                # 6,416 decompiled Java source files across 783 packages
├── 📁 resources/              # AndroidManifest.xml, layouts, drawables, strings, values
└── 📁 docs/research/          # Additional research documents
```

---

## 🗺️ Master Index & Navigation

> **Start here** → The two index files below catalog and cross-reference every analysis document in this repository.

| Index File | Purpose |
| :--- | :--- |
| [`tivimate_master_reference_catalog.md`](tivimate_master_reference_catalog.md) | **Primary index.** Categorized catalog of all 23+ analyses with detailed section-by-section breakdowns of what each file covers. |
| [`tivimate_reference_master_index.md`](tivimate_reference_master_index.md) | **Priority-ranked index.** Maps files by feature area, risk level, and suggested implementation phase. |

---

## 📚 Complete Analysis File Catalog

### 🚀 Core Performance & Startup

| File | Description |
| :--- | :--- |
| [`tivimate_startup_process_deep_dive.md`](tivimate_startup_process_deep_dive.md) | **Complete chronological startup sequence.** Step-by-step breakdown of TiViMate's sub-300ms "Zero-Lag" cold start — from OS process spawn, async SQLite WAL pre-warming, SharedPreferences stream lookup, parallel ExoPlayer buffering, shutter view masking, to deferred WorkManager sync. Includes exact timing Gantt chart. |
| [`tivimate_cold_start_engine_analysis.md`](tivimate_cold_start_engine_analysis.md) | **Cold start engine audit.** Why the app starts and renders video in under 300ms — what database calls are pre-warmed, what UI bindings are deferred, and what sync processes are throttled on startup. |
| [`tivimate_buffering_and_startup_masterclass.md`](tivimate_buffering_and_startup_masterclass.md) | **Buffering masterclass.** The 4 pillars of TiViMate's buffering architecture: aggressive LoadControl tuning (500ms playback trigger), OkHttp socket pooling, visual shutter masking, and silent watchdog recovery. Includes implementation blueprints. |
| [`tivimate_startup_vod_ui_analysis.md`](tivimate_startup_vod_ui_analysis.md) | **Startup + VOD + UI overlays.** Covers the zero-lag startup routine, SQLite FTS4 full-text search engine, VOD catalog aggregate loading, and overlay panel (Quick Actions & Channel Bar) design patterns. |

---

### 📡 EPG Guide & Grid Rendering

| File | Description |
| :--- | :--- |
| [`tivimate_guide_performance_analysis.md`](tivimate_guide_performance_analysis.md) | **EPG performance deep-dive.** How TiViMate achieves 60fps scrolling over thousands of channels — sorted RAM arrays, custom direct-to-Canvas pixel drawing, 90-minute TTL cache, two-table EPG schema, and timezone offset stacking. |
| [`tivimate_epg_sync_analysis.md`](tivimate_epg_sync_analysis.md) | **EPG synchronization workers.** Execution flow, background threading, and persistence mechanisms for EPG data sync. |

---

### 🎬 Playlist Ingestion & Parsing

| File | Description |
| :--- | :--- |
| [`tivimate_playlist_parsing_stalker_xc.md`](tivimate_playlist_parsing_stalker_xc.md) | **Multi-protocol playlist architecture.** M3U/M3U8 parsing rules, Xtream Codes 4-step API handshake, and Stalker Portal MAG box emulation with MAC-based auth. |
| [`tivimate_playlist_parser_analysis.md`](tivimate_playlist_parser_analysis.md) | **M3U parser optimizations.** Streaming parse loops, tokenization mechanisms, and `#EXTINF` metadata extraction strategies. |
| [`tivimate_playlist_management_analysis.md`](tivimate_playlist_management_analysis.md) | **Playlist processing & customization.** How M3U/XC sources map into relational SQLite schemas — favorites, soft-deletes, custom reordering, and manual EPG bindings. |
| [`tivimate_playlist_setup_wizard_deep_dive.md`](tivimate_playlist_setup_wizard_deep_dive.md) | **Setup wizard pipeline.** Step-by-step from initial UI interaction through multi-protocol validation, Room insertions, background sync, to ExoPlayer buffering and live video rendering. |
| [`tivimate_playlist_update_safety_analysis.md`](tivimate_playlist_update_safety_analysis.md) | **Update safety & state preservation.** Incremental diffing transactions, user customization protection during imports, and fault-tolerant sync error handling. |
| [`tivimate_playlist_epg_update_analysis.md`](tivimate_playlist_epg_update_analysis.md) | **Playlist & EPG sync engine.** WorkManager scheduling, interval configs, transactional data structures, and decompiled code references for reliable background updates. |

---

### 🌐 Stalker Portal & API Integration

| File | Description |
| :--- | :--- |
| [`tivimate_stalker_portal_deep_dive.md`](tivimate_stalker_portal_deep_dive.md) | **Complete Stalker Portal integration.** Exhaustive code-traced analysis covering MAC-based auth handshake, transient token lifecycle, VOD/Series lazy-loading, genre pagination, full-text search, and silent error-recovery engine. |
| [`tivimate_catchup_api_integration_analysis.md`](tivimate_catchup_api_integration_analysis.md) | **Catch-up & Portal API client.** Dynamic catch-up URL construction, state machine seek requests, and Stalker token auto-refresh routines. |

---

### 🎥 Playback, Streaming & Player Controls

| File | Description |
| :--- | :--- |
| [`tivimate_streaming_freeze_resync_deep_dive.md`](tivimate_streaming_freeze_resync_deep_dive.md) | **Streaming stability deep-dive.** Freeze detection, source rechecking, dynamic auth refreshes, and exponential backoff retry flows for absolute living-room reliability. |
| [`tivimate_network_player_tuning_analysis.md`](tivimate_network_player_tuning_analysis.md) | **Network stream tuning.** Custom ExoPlayer LoadControl (500ms/1000ms/2500ms), OkHttp socket pooling, and in-memory GZIP EPG decompression without disk wear. |
| [`tivimate_player_controls_analysis.md`](tivimate_player_controls_analysis.md) | **Custom player controls & track selection.** OSD overlay architecture and media track switching systems. |
| [`tivimate_playback_controls_settings_audit.md`](tivimate_playback_controls_settings_audit.md) | **Aspect ratio, audio sync & settings.** Layout matrix zooming (Normal/Fit/Fill/Stretch/Zoom), Media3 track overrides, audio offset correction, and slide-in Leanback settings drawers. |
| [`tivimate_external_player_pip_analysis.md`](tivimate_external_player_pip_analysis.md) | **External player & PiP.** How TiViMate offloads playback to VLC/MX Player via intents and implements Android Picture-in-Picture mode. |
| [`tivimate_multiview_architecture_analysis.md`](tivimate_multiview_architecture_analysis.md) | **Multiview (Multi Mode) engine.** Coordinating multiple concurrent ExoPlayer instances, hardware decoder limits, dynamic audio focus routing, and D-pad panel navigation. |

---

### ⏪ Catch-up, Timeshift & DVR

| File | Description |
| :--- | :--- |
| [`tivimate_catchup_controls_architecture.md`](tivimate_catchup_controls_architecture.md) | **Catch-up controls & timeshift.** EPG alignment, transport seek overlays, dynamic URL interpolation, and live-to-catchup transition mechanics. |
| [`tivimate_catchup_recording_reminders_analysis.md`](tivimate_catchup_recording_reminders_analysis.md) | **Catch-up, PVR & reminders.** Timeshift token interpolation, exact wake alarms, and background coordination conflict handling. |
| [`tivimate_pvr_recording_analysis.md`](tivimate_pvr_recording_analysis.md) | **PVR/DVR recording engine.** Background recording schedules, uninterrupted writes during TV sleep, stream drop recovery, and SMB network share integration. |
| [`tivimate_dvr_favorites_history_audit.md`](tivimate_dvr_favorites_history_audit.md) | **DVR, favorites & history.** CPU WakeLock lifecycle, double-buffered favorites indexing, and the 5-second surfing delay filter for intelligent history logging. |

---

### 📺 UI, OSD & Channel Navigation

| File | Description |
| :--- | :--- |
| [`tivimate_channelbar_actionsbar_infobar_deep_dive.md`](tivimate_channelbar_actionsbar_infobar_deep_dive.md) | **Channel/Actions/Info bar deep-dive.** Complete OSD overlay analysis — layout structure, animation pipelines, D-pad routing, visibility state machines, auto-hide timers, buffering indicators, and bar coordination. |
| [`tivimate_channel_change_osd_analysis.md`](tivimate_channel_change_osd_analysis.md) | **Channel zapping & OSD navigation.** Info Bar, Channel Bar, Quick Actions Bar layouts, D-pad interception, and performance optimizations for fluid couch UX. |
| [`tivimate_channel_group_customization_analysis.md`](tivimate_channel_group_customization_analysis.md) | **Channel & group customization.** Custom groups, row-action reordering, and parental blocks at the database and UI layer. |
| [`tivimate_transparency_video_overlay_analysis.md`](tivimate_transparency_video_overlay_analysis.md) | **Transparency & video overlay.** Layered alpha scrim rendering — live video behind EPG grids, search panels, and settings drawers without frame drops. |

---

### 📱 VOD (Movies & Series)

| File | Description |
| :--- | :--- |
| [`tivimate_vod_playback_actions_bar_analysis.md`](tivimate_vod_playback_actions_bar_analysis.md) | **VOD playback & actions bar.** Complete VOD pipeline — loading, playing, and controlling Movies & TV Shows — and how the player OSD differs between VOD and Live TV modes. |
| [`tivimate_vod_metadata_resume_analysis.md`](tivimate_vod_metadata_resume_analysis.md) | **VOD metadata & playback resume.** Layered season/episode structures, network metadata caching, and persistent playhead position storage. |

---

### 🔤 Subtitles

| File | Description |
| :--- | :--- |
| [`tivimate_subtitle_deep_dive.md`](tivimate_subtitle_deep_dive.md) | **Complete subtitle system.** Full pipeline from format detection → decoding → track selection → rendering for both Live TV and VOD content. |

---

### 🎨 Theme, Design & Visual Effects

| File | Description |
| :--- | :--- |
| [`tivimate_theme_effects_engine.md`](tivimate_theme_effects_engine.md) | **Theme engine & visual effects.** Dynamic color theme swapping without interrupting video, focus glows, shadow elevations, glassmorphism, and premium fade-out effects. |
| [`tivimate_design_ux_reference_guide.md`](tivimate_design_ux_reference_guide.md) | **Design & UX reference guide.** High-end UX principles, visual tokens, dark-mode styling, layout structures, and remote control timing rules for TV app development. |

---

### 🔧 Remote Control, Input & Menus

| File | Description |
| :--- | :--- |
| [`tivimate_remote_input_menu_architecture.md`](tivimate_remote_input_menu_architecture.md) | **Remote input & menu customization.** Hardware D-pad key mapping, long-press detection, Android focus path overrides, and dynamic sidebar/bottom menu layout customization. |

---

### 🔍 Search

| File | Description |
| :--- | :--- |
| [`tivimate_search_architecture_analysis.md`](tivimate_search_architecture_analysis.md) | **Search-as-you-type engine.** SQLite FTS4 virtual tables, sub-millisecond MATCH queries over 100K+ items, debounced input flow, and TV keyboard focus lock behavior. |

---

### 🗄️ Database, Schema & Migrations

| File | Description |
| :--- | :--- |
| [`tivimate_complete_database_schema_migrations.md`](tivimate_complete_database_schema_migrations.md) | **Complete SQLite schema reference.** Every table, column, and relationship in TiViMate's Room database — WAL config, migration framework, schema validation, and identity hash verification. |
| [`tivimate_settings_backup_database_analysis.md`](tivimate_settings_backup_database_analysis.md) | **Settings, backup & database.** Settings precedence layers, backup/restore internals, and relational schema map for indexing massive IPTV datasets. |

---

### 💾 Backup & Restore

| File | Description |
| :--- | :--- |
| [`tivimate_backup_restore_deep_dive.md`](tivimate_backup_restore_deep_dive.md) | **Backup & restore deep-dive.** SQLite WAL checkpointing, preference packaging, SAF/USB/SMB directory targets, compatibility verification, and safe database hot-swaps with rollback protection. |

---

### 🔐 Licensing, Security & Parental Controls

| File | Description |
| :--- | :--- |
| [`tivimate_licensing_device_management.md`](tivimate_licensing_device_management.md) | **Licensing & device management.** Device identification, premium account verification, 5-device limitation enforcement, and offline JWT token validation. |
| [`tivimate_parental_controls_security_analysis.md`](tivimate_parental_controls_security_analysis.md) | **Parental controls & security.** Hidden vs. blocked categories, secure PIN hash storage in SharedPreferences, and identified security weaknesses to avoid. |

---

### 🖼️ Logos & Channel Icons

| File | Description |
| :--- | :--- |
| [`tivimate_logo_resolution_pipeline.md`](tivimate_logo_resolution_pipeline.md) | **Logo resolution pipeline.** How TiViMate resolves, prioritizes, caches (250MB Glide disk cache), and renders channel and VOD logos/posters. |

---

### ⏰ Sleep Timer, Auto-Boot & Background Tasks

| File | Description |
| :--- | :--- |
| [`tivimate_sleep_timer_auto_boot_analysis.md`](tivimate_sleep_timer_auto_boot_analysis.md) | **Sleep timer & auto-boot lifecycle.** Countdown state management, clean app exit sequences, BOOT_COMPLETED auto-start, wake-from-sleep detection, and autoplay routing. |
| [`tivimate_workmanager_background_tasks.md`](tivimate_workmanager_background_tasks.md) | **WorkManager & background tasks.** WorkSpec table schema, task dependency chaining, periodic EPG/playlist sync scheduling, network constraints, exponential backoff retry, and stale job cleanup. |

---

### 🛡️ Error Handling & Device Optimization

| File | Description |
| :--- | :--- |
| [`tivimate_error_handling_low_end_device_analysis.md`](tivimate_error_handling_low_end_device_analysis.md) | **Low-end device optimization & error handling.** Memory/decoder strategies for restricted TV chipsets, fallback error taxonomies, stream crash recovery, and external player intent state transfers. |
| [`tivimate_auxiliary_systems_analysis.md`](tivimate_auxiliary_systems_analysis.md) | **Auxiliary systems.** Fuzzy logo matching, external player resume sync (VLC/MX Player playback position return), and standby sleep timer engine. |
| [`tivimate_advanced_features_analysis.md`](tivimate_advanced_features_analysis.md) | **Advanced features.** Modern SMBJ (SMBv2/v3) network share implementation, unified local/network folder browser, D-pad channel recall/history, and the Multiview rendering engine. |

---

### 📋 Guides, Inventories & Roadmaps

| File | Description |
| :--- | :--- |
| [`tivimate_usability_insights_guide.md`](tivimate_usability_insights_guide.md) | **Usability insights guide.** Separates clean, reusable architectural patterns from compiler-generated artifacts and obfuscated code you should NOT copy. |
| [`clean_room_behavior_inventory.md`](clean_room_behavior_inventory.md) | **Clean room behavior inventory.** Comprehensive behavioral catalog of TiViMate's observable features documented for clean-room reimplementation purposes. |
| [`purefusion_clean_room_playback_roadmap.md`](purefusion_clean_room_playback_roadmap.md) | **PureFusion clean room playback roadmap.** Implementation roadmap for building a clean-room playback engine inspired by TiViMate's architectural patterns. |
| [`purefusion_local_broadcast_epg_analysis.md`](purefusion_local_broadcast_epg_analysis.md) | **PureFusion local broadcast EPG analysis.** EPG integration strategy for local broadcast sources in the PureFusion project. |

---

### 📁 Additional Directories

| Directory | Contents |
| :--- | :--- |
| `sources/` | **6,416 decompiled `.java` files** across 783 packages — the complete TiViMate v5.2.0 decompiled source tree (JADX output). Obfuscated class names (e.g., `C4894.java`, `p411/`). |
| `resources/` | **AndroidManifest.xml**, 1,587 resource files — layouts, drawables, strings (`values/`, `values-fr/`), styles, and XML configs. |
| `docs/research/` | Extended research documents including the clean-room behavior inventory. |

---

## 🔑 Key Architectural Highlights

<details>
<summary><strong>⚡ Sub-300ms Cold Start</strong></summary>

TiViMate achieves near-instant startup by:
1. **Async SQLite pre-warming** with `PRAGMA journal_mode=WAL` in `Application.onCreate()`
2. **Instant SharedPreferences** fetch for `last_channel_url` (< 0.5ms)
3. **Parallel ExoPlayer init** — buffering starts before UI inflates
4. **500ms playback trigger** via custom `LoadControl` (vs. default 2,500ms)
5. **30-60s deferred background sync** — WorkManager tasks sleep during boot
</details>

<details>
<summary><strong>📺 60fps EPG Grid</strong></summary>

No `RecyclerView` — TiViMate draws directly to Android `Canvas` using pixel-coordinate math, avoiding view inflation, layout passes, and garbage collection entirely.
</details>

<details>
<summary><strong>🔌 Triple Protocol Support</strong></summary>

- **M3U/M3U8** — Streaming tokenized parser with `#EXTINF` metadata extraction
- **Xtream Codes** — Full 4-step API (auth → categories → streams → EPG)
- **Stalker Portal** — Complete MAG box emulation with MAC auth, token refresh, and genre pagination
</details>

<details>
<summary><strong>🎬 Visual Shutter Masking</strong></summary>

A black overlay (`f14919`) fades in immediately on channel change, hiding decoder artifacts (P/B frame garbage). It fades out smoothly (150ms) only after `onRenderedFirstFrame()` confirms a clean keyframe is rendered.
</details>

---

## ⚠️ Disclaimer

This repository contains **architectural analysis and documentation only** derived from publicly available decompiled code for **educational and research purposes**. No proprietary source code is redistributed in executable form. All trademarks belong to their respective owners.

---

<p align="center">
  <sub>📝 Analysis by <a href="https://github.com/Eliminater74">Eliminater74</a> • Built with ☕ and reverse engineering</sub>
</p>
