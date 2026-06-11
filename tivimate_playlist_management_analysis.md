# TiViMate Playlist Processing & Customization — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s playlist processing, caching, and editing engine**. It dissects how M3U files and Xtream Codes APIs are mapped into relational SQLite schemas, and details the database transactions behind favorite channels, soft-deletes, custom reordering index shifts, and manual EPG source mapping.

---

## 1. THE RELATIONAL PLAYLIST SCHEMA SYSTEM

To make massive lists of up to 100,000 streams browseable in microseconds, TiViMate avoids flat JSON or memory files. It structures all imports inside an interconnected, indexed Room database.

```
                  ┌──────────────┐
                  │  playlists   │
                  └──────┬───────┘
                         │ (1-to-many)
            ┌────────────┴────────────┐
            ▼                         ▼
   ┌─────────────────┐       ┌─────────────────┐
   │   categories    │       │  tvg_sources    │
   └────────┬────────┘       └────────┬────────┘
            │ (1-to-many)             │ (1-to-many)
   ┌────────▼────────┐       ┌────────▼────────┐
   │    channels     │       │  tvg_channels   │
   └────────┬────────┘       └────────┬────────┘
            │                         │
            └───────────┬─────────────┘
                        ▼
         ┌──────────────────────────────┐
         │    channel_tvg_bindings      │ (Maps channel_id to tvg_channel_id)
         └──────────────────────────────┘
```

### The Core Relational Schema

*   **`playlists`**: Holds the portal/file details, credentials, user-agents, catchup types, and transient session credentials (`stalker_token` or XC cookies).
*   **`channels`**: Relates to a `category_id` and `playlist_id`. Stores the raw streaming URL, custom name overrides, stream indexes, and status flags.
*   **`channel_tvg_bindings`**: A junction table that maps `channel_id` (from playlists) directly to `tvg_channel_id` (from external EPG providers).

---

## 2. EDITING, CUSTOMIZING & CUSTOM ORDER INDEX SHIFTS

When a user manages their playlist in TiViMate (hiding channels, reordering categories, adding favorites, or renaming streams), the modifications are written instantly to SQLite using transaction-guarded queries.

### A. Soft-Deletes & Visibility Toggles (Hiding)
*   **The Design Secret**: When a user clicks "Hide Group" or "Hide Channel", the application **never** runs a destructive `DELETE` query. Doing so would lose user custom names, historical PVR records, or favorite states if the user decides to un-hide the stream later.
*   **Decompiled Formula**:
    The database flags soft deletes by writing timestamps to a `deleted_time` column or toggling an `is_visible` integer flag. 
    Queries then look up active items by checking:
    ```sql
    WHERE c.deleted_time IS NULL 
      AND IFNULL(c.is_visible, 1) == 1
    ```
    This separates temporary UI hidden states from active relational storage.

### B. Custom Reordering Index Shifts (Moving Channels/Groups)
*   **The Design Secret**: Moving items up/down in a remote-controlled list requires a solid reindexing system. Categories and channels use index position columns (`position` and `position_in_playlist`).
*   **Decompiled Formula**:
    When a user moves an item from index A to index B, the app does not rewrite all rows. It triggers a bulk index shift inside a SQLite transaction:
    1.  **Shift Downwards** (to make room):
        ```sql
        UPDATE channels 
        SET position_in_playlist = position_in_playlist + 1 
        WHERE playlist_id == ? 
          AND category_id == ?
          AND position_in_playlist >= ? -- target index B
        ```
    2.  **Insert Item** into target slot B:
        ```sql
        UPDATE channels 
        SET position_in_playlist = ? -- target B 
        WHERE id == ?
        ```
    This is executed within `database.withTransaction {}` to keep the database locked for only a few milliseconds, ensuring the EPG UI grid updates dynamically without flashing black.

### C. Custom Names & Overrides (Renaming)
*   **The Design Secret**: Users can assign custom aliases to channels and category headers.
*   **Decompiled Formula**:
    The `channels` and `movie_categories` tables feature dedicated override columns: `custom_name` and `custom_group_name`.
    *   *Renaming query inside `C3220.java` (Line 93)*:
        ```sql
        UPDATE channels SET custom_name = ? WHERE id == ?
        ```
    *   *UI rendering logic*:
        The custom view resolver queries both columns:
        ```kotlin
        val displayName = channel.customName ?: channel.rawName
        ```
        This ensures raw M3U playlist updates never overwrite your custom channel labels.

### D. Decoupled EPG Source Bindings
*   **The Design Secret**: If an M3U playlist doesn't have accurate EPG associations, the user can manually bind EPG programs.
*   **Decompiled Formula**:
    Instead of editing the raw M3U file, manual EPG bindings are recorded in the `channel_tvg_bindings` junction table. 
    When an EPG sync task runs in the background, it joins this table to fetch EPG program records dynamically, keeping EPG associations entirely decoupled from physical playlist updates.

---

## 3. HOW YOU CAN APPLY THESE PARADIGMS

### 🟢 What You CAN Use & Adapt:
*   **Soft Delete Filters**:
    Always include a nullable `deleted_time` or `is_visible` flag in your Room entities. Apply filter check bounds on all database queries:
    ```kotlin
    @Query("SELECT * FROM channels WHERE category_id = :catId AND deleted_time IS NULL ORDER BY position_in_playlist ASC")
    fun getActiveChannelsForCategory(catId: Long): List<ChannelEntity>
    ```
*   **Atomic Index Shifts**:
    Write reordering functions inside atomic transaction blocks:
    ```kotlin
    suspend fun moveChannel(channelId: Long, categoryId: Long, fromPos: Int, toPos: Int) = roomDatabase.withTransaction {
        if (fromPos < toPos) {
            // Shift intermediary channels up
            channelDao.decrementPositions(categoryId, fromPos + 1, toPos)
        } else {
            // Shift intermediary channels down
            channelDao.incrementPositions(categoryId, toPos, fromPos - 1)
        }
        channelDao.updatePosition(channelId, toPos)
    }
    ```
*   **Junction Tables for Custom Bindings**:
    Maintain EPG source bindings in a dedicated relational mapping table rather than string-manipulating M3U stream inputs.

### 🔴 What You CANNOT Copy:
*   **Obfuscated Query Wrappers**:
    Avoid hardcoded database mapping index columns from decompiled files like `C3210.java` or `C3220.java`. These contain obfuscated query strings generated by the R8 packaging compiler. Write clean Room DAO queries.
