# TiViMate Settings, Backup & Database Schema — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s settings precedence layers, backup/restore internals, and relational SQLite schema map**. It dissects how the application overrides global configurations at the channel level, packages database and preference files into compressed backup archives, and indexes massive IPTV datasets for microsecond search queries.

---

## 1. SETTINGS PRECEDENCE & CASCADING OVERRIDES

To provide maximum flexibility, TiViMate allows users to configure settings globally, but override them for a specific playlist, a particular category, or even an individual channel.

### The Cascading Precedence Pipeline

```
 [ Global App Defaults ] ──► [ Playlist Settings Override ] ──► [ Category Override ] ──► [ Channel Specific Override ]
```

When rendering streams or determining catch-up limits, the application resolves parameters using a **cascading fallback logic**:

```kotlin
// Replicated safe precedence resolution
fun getEffectiveCatchupHours(channel: Channel, category: Category, playlist: Playlist): Int {
    return channel.catchupHours 
        ?: category.catchupHours 
        ?: playlist.catchupHours 
        ?: GLOBAL_DEFAULT_CATCHUP_HOURS
}
```

### Storage Implementation
*   **Global Preferences**: Stored inside Android's default `SharedPreferences` XML file.
*   **Scoped Overrides**: Recorded as nullable columns inside the SQLite relational tables. The database tables `playlists`, `categories`, and `channels` all host parallel columns (e.g. `catchup_hours`, `user_catchup_type`, `logos_priority`). If a column value is `NULL`, the app cascades down to the next scope.

---

## 2. BACKUP & RESTORE INTERNALS (DATA SAFETY ENGINE)

To move settings between TV boxes, TiViMate generates a compressed backup file (`.tmb` extension).

### The In-Memory Packaging Architecture
A TiViMate backup is a **compressed GZIP archive** containing:
1.  **`TvPlayer.db`**: The complete active SQLite database file.
2.  **`settings.xml`**: A copy of the application's SharedPreferences XML file.
3.  **`metadata.json`**: A header holding the app's version code, build name, timestamp, and an MD5 checksum to verify data integrity.

### The Safe Restore Sequence

```
[ Verify Backup Header ] ──► [ Verify MD5 Checksum ] ──► [ Close active SQLite WAL connections ]
                                                                      │
[ Restart App Lifecycle ] ◄─── Swap database files ◄─── Copy old DB to temp.bak (Fallback)
```

1.  **Version Validation**: The restore manager reads the backup metadata first. If the backup's version code is *higher* than the active app's version code, it blocks the restore to prevent schema downgrade crashes.
2.  **Checksum Check**: It verifies the archive's MD5 checksum. If corrupt, it aborts immediately.
3.  **Database Connection Closing**: Standard Room database connections are closed: `database.close()`. This stops background EPG sync threads.
4.  **Fail-safe Swap**:
    *   Renames the active `TvPlayer.db` to `TvPlayer.db.bak` (temporary fallback).
    *   Extracts the restored database file to `TvPlayer.db`.
    *   If extraction fails, it restores the `.bak` file instantly.
5.  **Force Reboot**: The app calls `Process.killProcess(Process.myPid())` or triggers a custom restart pending intent. This forces the Android OS to launch a fresh process, rebuilding the Room SQLite instance cleanly under the restored data.

---

## 3. DATABASE SCHEMA & PERFORMANCE INDEX MAP

The database is built using a normalized SQLite WAL model. Complex searches and EPG rendering use pre-compiled composite indexes and FTS virtual tables.

```
       ┌─────────────────────────────────────────────────────────┐
       │                 TvPlayer Database Map                  │
       └─────────────────────────────────────────────────────────┘
       
   [ channels ]  ────────────── Composite Index ──────────────► [ idx_channels_playlist_cat ]
     (Relational data)                                            (Optimizes TV Guide vertical scrolling)
       │
       ├──────────────► [ tvg_programs ] ──► [ idx_tvg_programs_channel_time ]
       │                                     (Optimizes horizontal timeline seeking)
       │
       └──────────────► [ tvg_programs_fts ] (FTS4 Virtual Search Index)
```

### Performance-Critical Indexes

To maintain 60fps guide scrolling over 50,000 items, SQLite is optimized via composite B-tree indexes:

*   **EPG Timeline Search Index**:
    ```sql
    CREATE INDEX IF NOT EXISTS idx_tvg_programs_channel_time 
    ON tvg_programs (tvg_channel_id, start_epoch_seconds);
    ```
    *   *Why this matters*: When binary-searching EPG programs, this index allows SQLite to retrieve matching start times in microseconds without scanning the entire database.
*   **Playlist-Category Filter Index**:
    ```sql
    CREATE INDEX IF NOT EXISTS idx_channels_playlist_cat 
    ON channels (playlist_id, category_id, position_in_playlist);
    ```
    *   *Why this matters*: Optimizes category switching, rendering only visible channel rows instantly.

### Full-Text Search (FTS4) Integration
EPG search bypasses standard relational tables, routing string searches directly to parallel virtual search tables (`tvg_programs_fts`):

```sql
CREATE VIRTUAL TABLE tvg_programs_fts USING fts4(
    content="tvg_programs", 
    title, 
    desc
);
```

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Rename your database columns to match exact Epoch Seconds (`INTEGER` in SQLite).
*   Add composite indexes `idx_tvg_programs_channel_time` onto your EPG database tables.

### Medium-Risk Actions
*   Build a sliding Preference Drawer overlay in Kotlin using dynamic styles to adjust fonts and grid density dynamically on-the-fly.
*   Implement cascading settings resolution models inside your repository classes.

### High-Risk Actions
*   Performing raw database swaps during active write processes without cleanly closing Room database connections first. Always invoke `database.close()` and shut down background executors before running restores.

### What to Measure First
*   Benchmark database query durations. EPG viewport queries must conclude in **under 3ms** to sustain a fluid 60fps scrolling frame rate.

### What Not to Copy Directly
*   Do not copy obfuscated Room helper methods or compiled transaction bindings from `C3214.java`.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Database index performance tuning and a bulletproof backup/restore manager are essential to prevent data corruption.
