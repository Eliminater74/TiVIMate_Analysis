# TiViMate Complete SQLite Database Schema & Migration Architecture

This document provides a definitive, complete reference of **every SQLite table, column, and relationship** in TiViMate's Room database, including the WAL configuration, migration framework, schema validation, and identity hash verification system. All data is extracted directly from decompiled DAO classes (`C3210.java`, `C3214.java`), the database helper (`C1233.java`), and SQL string analysis.

---

## 1. DATABASE INITIALIZATION & WAL CONFIGURATION

### 1.1 PRAGMA Configuration Pipeline

TiViMate configures SQLite with high-performance settings at database open time. This happens in `C1233.java` method `m3811`:

```sql
-- Step 1: Journal Mode (WAL for concurrent read/write)
PRAGMA journal_mode = WAL

-- Step 2: Synchronous Mode (NORMAL = faster writes, safe with WAL)
PRAGMA synchronous = NORMAL

-- Step 3: Busy Timeout (3 second retry on lock contention)
PRAGMA busy_timeout = 3000
```

**Source Evidence** (`C1233.java` lines 175-185):
```java
if (c1247.f4849 == 3) {
    exec(interfaceC4932, "PRAGMA journal_mode = WAL");
} else {
    exec(interfaceC4932, "PRAGMA journal_mode = TRUNCATE");
}
if (c1247.f4849 == 3) {
    exec(interfaceC4932, "PRAGMA synchronous = NORMAL");
} else {
    exec(interfaceC4932, "PRAGMA synchronous = FULL");
}
```

The busy timeout check (`C1233.java` line 157):
```java
if (j < 3000) {
    exec(interfaceC4932, "PRAGMA busy_timeout = 3000");
}
```

### 1.2 Room Identity Hash Verification

TiViMate uses Android Room's `room_master_table` to verify database schema integrity on every open:

```sql
CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY, identity_hash TEXT)
INSERT OR REPLACE INTO room_master_table (id, identity_hash) VALUES(42, '<hash>')
```

On subsequent opens, it validates:
```sql
SELECT identity_hash FROM room_master_table WHERE id = 42 LIMIT 1
```

If the hash mismatches, Room throws: *"Looks like you've changed schema but forgot to update the version number."*

---

## 2. COMPLETE TABLE REFERENCE

### 2.1 `playlists` — Master Playlist Registry

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Auto-increment primary key |
| `name` | TEXT | User-visible playlist name |
| `url` | TEXT | Playlist URL (M3U/XC/Stalker endpoint) |
| `is_vod` | BOOLEAN | `0` = TV playlist, `1` = VOD-only playlist |
| `is_enabled` | BOOLEAN | Whether playlist is active |
| `auto_update` | BOOLEAN | Auto-update on interval |
| `update_interval_hours` | INT | Hours between auto-updates |
| `last_update_time` | LONG | Epoch of last successful update |
| `include_tv_channels` | BOOLEAN | Include live TV channels (XC playlists) |
| `include_vod` | BOOLEAN | Include VOD content (XC playlists) |
| `is_expanded` | BOOLEAN | UI expansion state in channel groups |
| `is_expanded_in_movies` | BOOLEAN | UI expansion state in movie groups |
| `is_expanded_in_series` | BOOLEAN | UI expansion state in series groups |
| `tvg_urls` | TEXT | Semicolon-separated EPG source URLs |
| `tvg_shift` | INT | EPG timezone shift offset |
| `server_timezone` | TEXT (nullable) | Server timezone from XC API |
| `expiration_date` | LONG (nullable) | Account expiration epoch |
| `max_connections` | INT (nullable) | Server connection limit |
| `channel_count` | INT | Cached count of channels |
| `movie_count` | INT | Cached count of movies |
| `series_count` | INT | Cached count of series |
| `catchup_type` | INT (nullable) | Default catch-up format (0=None, 1=XC, 2=Flussonic, 3=Append) |
| `catchup_hours` | INT (nullable) | Catch-up archive depth in hours |
| `catchup_source` | TEXT (nullable) | Catch-up URL template |
| `user_catchup_type` | INT (nullable) | User-overridden catch-up type |
| `user_catchup_hours` | INT (nullable) | User-overridden catch-up hours |
| `user_catchup_time_offset` | INT (nullable) | Manual timezone correction offset |
| `is_visible_in_all_channels` | BOOLEAN | Show in "All channels" virtual group |
| `is_visible_in_all_favorites` | BOOLEAN | Show in "All favorites" virtual group |
| `logos_priority` | INT | Logo source priority enum |
| `prev_groups_sorting` | INT | Previous channel group sort mode (for undo) |
| `groups_sorting` | INT | Current channel group sort mode |
| `prev_movie_groups_sorting` | INT | Previous movie group sort mode |
| `movie_groups_sorting` | INT | Current movie group sort mode |
| `prev_series_groups_sorting` | INT | Previous series group sort mode |
| `series_groups_sorting` | INT | Current series group sort mode |
| `position` | INT (nullable) | Manual playlist ordering position |
| `stalker_token` | TEXT (nullable) | Stalker Portal auth token |
| `user_agent` | TEXT (nullable) | Custom HTTP User-Agent header |
| `are_new_groups_visible` | BOOLEAN | Auto-show newly added channel groups |
| `are_new_movie_groups_visible` | BOOLEAN | Auto-show newly added movie groups |
| `are_new_series_groups_visible` | BOOLEAN | Auto-show newly added series groups |

---

### 2.2 `channels` — Live TV Channel Items

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Auto-increment primary key |
| `playlist_id` | LONG (FK → playlists) | Parent playlist |
| `original_group_id` | LONG (FK → channel_groups) | Original group assignment |
| `xc_id` | INT (nullable) | Xtream Codes stream ID |
| `stalker_id` | LONG (nullable) | Stalker Portal channel ID |
| `stalker_genre_id` | LONG (nullable) | Stalker genre binding |
| `stalker_type` | TEXT (nullable) | Stalker content type |
| `name` | TEXT | Channel name from playlist |
| `custom_name` | TEXT (nullable) | User-renamed channel name |
| `url` | TEXT | Stream URL |
| `logo` | BLOB (nullable) | Logo URL from playlist (stored as bytes) |
| `position_in_playlist` | INT | Original position in M3U file |
| `is_visible` | BOOLEAN | Visibility in channel list |
| `is_favorite` | BOOLEAN (nullable) | Favorited by user |
| `display_mode` | INT | Aspect ratio mode (1=Normal, 2=Fill, 3=Zoom, 4=Stretch) |
| `audio_offset` | INT (nullable) | Audio sync correction in milliseconds |
| `watch_time` | LONG | Total watch time in seconds |
| `last_turn_on_time` | LONG (nullable) | Last viewed epoch (for "Recent channels") |
| `catchup_hours` | INT (nullable) | Per-channel catch-up override |
| `deleted_time` | LONG (nullable) | Soft-delete timestamp |

---

### 2.3 `channel_groups` — Channel Category Containers

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `playlist_id` | LONG (FK → playlists) | Parent playlist |
| `name` | TEXT | Group name from `group-title` M3U tag |
| `is_custom` | BOOLEAN | `true` = user-created custom group |
| `deleted_time` | LONG (nullable) | Soft-delete timestamp |

---

### 2.4 `channel_group_options` — Per-Group User Customizations

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `group_id` | LONG (FK → channel_groups) | Parent group |
| `custom_group_name` | TEXT (nullable) | User-renamed group name |
| `is_visible` | BOOLEAN (nullable) | Group visibility override |
| `position` | INT (nullable) | Manual sort position |

---

### 2.5 `channel_manual_positions` — Per-Channel Sort Overrides

Used when users manually drag-and-drop channels within a group.

---

### 2.6 `channel_tvg_bindings` — EPG Channel Mappings

| Column | Type | Description |
|---|---|---|
| `channel_id` | LONG (FK → channels) | Bound channel |
| `tvg_channel_id` | LONG (FK → tvg_channels) | Matched EPG channel |
| `is_tvg_updated` | BOOLEAN | Whether EPG data has been refreshed |

---

### 2.7 `tvg_channels` — EPG Channel Registry

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `tvg_source_id` | LONG (nullable, FK → tvg_sources) | EPG source |
| `xmltv_channel_id` | TEXT | XMLTV channel identifier |
| `icon_url` | TEXT | Channel logo URL from EPG |

---

### 2.8 `tvg_sources` — EPG Source Registry

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `url` | TEXT | EPG XMLTV source URL |

---

### 2.9 `tvg_programs` — EPG Program Listings

Stores individual program entries parsed from XMLTV data. Includes FTS virtual table `tvg_programs_fts` for instant program title search.

---

### 2.10 `movies` — VOD Movie Items

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `playlist_id` | LONG (FK → playlists) | Parent playlist |
| `category_id` | LONG (nullable, FK → movie_categories) | Movie category |
| `xc_id` | INT (nullable) | Xtream Codes VOD ID |
| `stalker_id` | LONG (nullable) | Stalker Portal VOD ID |
| `name` | TEXT | Movie title |
| `image` | BLOB (nullable) | Poster URL (stored as bytes) |
| `rating` | FLOAT (nullable) | Rating (0.0-10.0) |
| `added_time` | LONG (nullable) | When movie was added to catalog |
| `position_in_playlist` | INT (nullable) | Order in provider catalog |
| `is_favorite` | BOOLEAN (nullable) | In user's "My List" |
| `last_turn_on_time` | LONG (nullable) | Last watched epoch (for history) |
| `display_mode` | INT | Aspect ratio mode |
| `deleted_time` | LONG (nullable) | Soft-delete timestamp |

**FTS Index:** `movies_fts` virtual table for instant title search via `MATCH` operator.

---

### 2.11 `movie_categories` — Movie Genre/Category Groups

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `playlist_id` | LONG (FK → playlists) | Parent playlist |
| `name` | TEXT | Category name (special: `__all_vod_category_name__`) |
| `is_visible` | BOOLEAN | Visibility toggle |
| `position` | INT (nullable) | Sort position |
| `stalker_id` | LONG (nullable) | Stalker Portal category ID |
| `stalker_genre_id` | LONG (nullable) | Stalker genre ID |
| `stalker_type` | TEXT (nullable) | Stalker content type classifier |
| `item_count` | INT (nullable) | Cached movie count in category |
| `deleted_time` | LONG (nullable) | Soft-delete timestamp |

---

### 2.12 `series` — TV Show Containers

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `playlist_id` | LONG (FK → playlists) | Parent playlist |
| `category_id` | LONG (nullable, FK → series_categories) | Show category |
| `xc_id` | INT (nullable) | Xtream Codes series ID |
| `stalker_id` | LONG (nullable) | Stalker Portal series ID |
| `name` | TEXT | Series title |
| `image` | BLOB (nullable) | Poster URL |
| `rating` | FLOAT (nullable) | Rating |
| `last_modified_time` | LONG (nullable) | Last catalog modification |
| `position_in_playlist` | INT (nullable) | Provider sort order |
| `is_favorite` | BOOLEAN (nullable) | In user's "My List" |
| `deleted_time` | LONG (nullable) | Soft-delete timestamp |

**FTS Index:** `series_fts` virtual table for instant title search.

---

### 2.13 `series_categories` — Show Genre/Category Groups

Same structure as `movie_categories` but for TV shows.

---

### 2.14 `episodes` — Individual Episode Items

Contains episode stream URL, season/series binding, title, duration, sequence number.

---

### 2.15 `recordings` — PVR Recording Registry

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `channel_id` | LONG (FK → channels) | Recorded channel |
| `status` | INT | 0=Scheduled, 1=Recording, 2=Failed, 3=Completed |
| `start_seconds` | LONG | Recording start epoch |
| `stop_seconds` | LONG | Recording stop epoch |
| `file_path` | TEXT | Output file path |
| `original_channel_name` | TEXT | Channel name at recording time |
| `program_start_seconds` | LONG (nullable) | EPG program start |
| `program_stop_seconds` | LONG (nullable) | EPG program stop |
| `program_title` | TEXT (nullable) | EPG program title |
| `program_desc` | TEXT (nullable) | EPG program description |
| `custom_recording_name` | TEXT (nullable) | User-assigned name |
| `repeat_rules` | TEXT (nullable) | Repeat schedule rules |

**Source Evidence** (`C3214.java` line 387, case 20 — `RecordingForPush` entity).

---

### 2.16 `my_programs` — User's Saved Programs ("My List" for EPG)

| Column | Type | Description |
|---|---|---|
| `id` | LONG (PK) | Primary key |
| `channel_id` | LONG (FK → channels) | Associated channel |
| `program_start_seconds` | LONG | Program start epoch |
| `program_stop_seconds` | LONG | Program stop epoch |
| `program_title` | TEXT | Program title |

---

### 2.17 `history_programs` — EPG Viewing History

Tracks which EPG programs the user has watched. Cleared with `DELETE FROM history_programs`.

---

### 2.18 `Preference` — Key-Value Settings Store

| Column | Type | Description |
|---|---|---|
| `key` | TEXT (PK) | Setting identifier |
| `long_value` | LONG (nullable) | Numeric preference value |

---

### 2.19 WorkManager Tables (Background Job Scheduling)

| Table | Purpose |
|---|---|
| `workspec` | Individual work item definitions (playlist updates, EPG syncs) |
| `dependency` | Work item dependency chains |
| `workname` | Named unique work bindings |
| `worktag` | Tags for work item grouping |
| `SystemIdInfo` | System alarm/job scheduler IDs |
| `WorkProgress` | In-progress work status |

---

## 3. RELATIONSHIP DIAGRAM

```
playlists ──────┬──── channels ──────── channel_tvg_bindings ──── tvg_channels ──── tvg_sources
                │         │
                │         ├──── channel_groups ──── channel_group_options
                │         │                         channel_manual_positions
                │         │
                │         └──── recordings
                │
                ├──── movies ──────── movie_categories
                │         │
                │         └──── movies_fts (FTS4 virtual)
                │
                ├──── series ──────── series_categories
                │         │
                │         ├──── seasons
                │         │        └──── episodes
                │         │
                │         └──── series_fts (FTS4 virtual)
                │
                └──── my_programs
                      history_programs
```

---

## 4. MIGRATION FRAMEWORK

### 4.1 Version-Based Migration Engine

TiViMate uses Room's standard migration framework (`C1233.java` lines 320-496). The migration engine:

1. Reads current schema version: `PRAGMA user_version`
2. Finds applicable migration path from current → target version
3. Executes migrations inside `BEGIN EXCLUSIVE TRANSACTION`
4. Validates schema after migration: `mo3725()` (destructive validation)
5. Updates version: `PRAGMA user_version = <target>`

If no migration path exists and destructive migration is allowed, it drops all tables and recreates:
```sql
SELECT name, type FROM sqlite_master WHERE type = 'table' OR type = 'view'
DROP TABLE IF EXISTS <table>
DROP VIEW IF EXISTS <view>
```

### 4.2 Bulk Reset Operations

The app provides several bulk-reset SQL operations for maintenance:

| Operation | SQL | Source |
|---|---|---|
| Reset all display modes | `UPDATE channels SET display_mode = 1` | C3210 line 800 |
| Reset all audio offsets | `UPDATE channels SET audio_offset = NULL` | C3210 line 819 |
| Reset all watch times | `UPDATE channels SET watch_time = 0` | C3210 line 827 |
| Reset all movie display modes | `UPDATE movies SET display_mode = 1` | C3210 line 1006 |
| Clear movie history | `UPDATE movies SET last_turn_on_time = NULL WHERE last_turn_on_time > 0` | C3210 line 1016 |
| Clear saved programs | `DELETE FROM my_programs` | C3210 line 1024 |
| Clear viewing history | `DELETE FROM history_programs` | C3210 line 930 |
| Reset EPG bindings flag | `UPDATE channel_tvg_bindings SET is_tvg_updated = 0` | C3210 line 882 |
| Delete all EPG bindings | `DELETE FROM channel_tvg_bindings` | C3210 line 897 |

---

## 5. SOFT-DELETE PATTERN

TiViMate uses a **soft-delete pattern** across all content tables. Instead of `DELETE FROM`, items are marked with a `deleted_time` timestamp:

```sql
WHERE m.deleted_time IS NULL  -- Only show non-deleted items
WHERE c.deleted_time IS NULL  -- Filter out deleted channels
```

This enables:
* **Undo operations** — Items can be restored by clearing `deleted_time`
* **Safe playlist updates** — Old items are marked deleted, new ones inserted, avoiding data loss during network failures
* **Audit trail** — Knowing when items were removed
