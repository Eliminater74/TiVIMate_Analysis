# TiViMate VOD Metadata & Playback Resume — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s Video-on-Demand (VOD) catalog schema, TV series metadata engine, and playback resume marker persistence**. It explains how the app structures layered season/episode lists, caches network metadata, and persists playhead positions securely.

---

## 1. NORMALISED VOD & TV SHOWS SCHEMA

VOD TV Shows are highly complex due to hierarchical relationships (Series -> Seasons -> Episodes). TiViMate maps these cleanly in SQLite to enable rapid D-pad browsing across thousands of episodes.

```
       ┌────────────────────────┐
       │         series         │ (Holds Series title, plot, rating, poster URL)
       └───────────┬────────────┘
                   │ (1-to-many)
       ┌───────────▼────────────┐
       │         seasons        │ (Season Index, e.g. Season 1, Season 2)
       └───────────┬────────────┘
                   │ (1-to-many)
       ┌───────────▼────────────┐
       │        episodes        │ (Episode number, title, plot, stream URL, duration)
       └────────────────────────┘
```

### The Normalized Series Hierarchy

*   **`series`**: The root container. Stores the title, plot, rating, genres, custom favorite status, and the network poster path.
*   **`seasons`**: A simple structure relating to the parent series. Keyed via `series_id` and storing the `season_number`.
*   **`episodes`**: Relates to both the parent series and season. Keys: `series_id`, `season_id`. Stores the episode title, stream URL, sequence number, and duration.

---

## 2. PLAYBACK RESUME POSITION MARKERS

To match the performance of high-end streaming platforms (Netflix, Plex), a TV player must remember exactly where the user stopped watching a movie or TV episode.

### The Playback Position Lifecycle

```
 [ Video Playback Active ] ──► [ Background Coroutine Scheduler (10s) ]
                                              │
                                              ▼ (Write current milliseconds)
 [ SQLite Db: updatePlaybackMarker() ] ◄──────┴────── [ User stops stream / clicks Back ]
                                              │
                                              ▼ (Start stream next time)
 [ Prompt Dialog card ] ──► "Resume from 01:24:10? [Resume] [Start Over]"
```

1.  **Direct Playhead Tracking**:
    During active playback of a movie or episode, the media controller queries ExoPlayer's current position (`player.currentPosition`) in milliseconds.
2.  **Thread-Safe DB Flusher**:
    To prevent data loss in case of a power outage or crash, a background scheduler writes the position marker to the database every **10 seconds**:
    ```sql
    UPDATE movies SET resume_position_ms = ? WHERE id == ?
    ```
    *   *Safe bounds*: If the user stops within the first 10 seconds or last 95% of the video, the resume marker is ignored or cleared to avoid prompting resume overlays for credits.
3.  **The Resume Overlay Prompt**:
    When video playback is initiated next time, the player queries:
    ```sql
    SELECT resume_position_ms FROM movies WHERE id == ?
    ```
    If `resume_position_ms > 0`, the player stops the decoder buffer start, overlays a transparent dialog card: *"Resume from 01:24:10? [Resume] [Start Over]"*, and seeks the player source to the saved milliseconds if "Resume" is clicked.

---

## 3. METADATA LOADING & POSTER CACHING

*   **Metadata Caching**:
    IPTV providers serve VOD metadata via APIs `/player_api.php?action=get_vod_info`. Querying this API every time the user clicks a movie page creates immense network lag. TiViMate queries it once, writes the JSON body into a local `movies_metadata` table, and serves it instantly from local RAM on consecutive clicks.
*   **Poster Cache Tuning**:
    Movie posters are cached aggressively via **Glide's LRU internal cache factory** using custom disk size bounds (250MB). Poster downloads are prioritized via low network queues to prevent scrolling latency.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Design custom layouts for movie metadata pages using HSL glassmorphism drawers.
*   Configure Glide to allocate a large internal disk cache folder dedicated entirely to poster assets.

### Medium-Risk Actions
*   Implement playhead position tracking inside your media player layer, saving the position in milliseconds to SQLite every 10 seconds.
*   Clear the saved playback marker if the user stream reaches the 95% completion threshold.

### High-Risk Actions
*   Writing playhead position updates to SQLite on the main UI thread during video rendering, which causes immediate scrolling lag and micro-stutters. Always use `Dispatchers.IO` transactions for position flushing.

### What to Measure First
*   Benchmark resume prompt latency. Resolving the playhead marker and prompting the dialog must complete in **under 150ms** upon clicking "Play."

### What Not to Copy Directly
*   Do not copy obfuscated series parsing loops or format mapping adapters from `C3214.java`.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Resilient playhead resume trackers and a normalized Series -> Seasons -> Episodes database structure.
