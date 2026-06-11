# TiViMate Playlist Update Safety & State Preservation — Reference Analysis

This document provides a highly detailed architectural analysis of **TiViMate's playlist update engine and data preservation systems**. It explains how background sync tasks execute massive channel imports incrementally, protect custom user configurations (favorites, custom names, and manual EPG bindings) during refreshes, and prevent database corruption or data loss when server updates fail.

---

## 1. INCREMENTAL SYNCING & DATABASE TRANSACTIONS

Traditional IPTV applications suffer from "playlist wipes." When updating a playlist, they delete the entire category and channel tables and reload them from scratch. This creates massive scrolling stutters, blocks active players, and destroys database relationships.

TiViMate implements an **Incremental Diffing Sync Engine** wrapped in atomic SQLite transactions.

```
 [ Fetch incoming playlist stream ] ──► [ Diff against local SQLite cache ]
                                                     │
 ┌───────────────────────────────────────────────────┼───────────────────────────────────────────────────┐
 ▼ (Change detected: UPDATE)                         ▼ (New item: INSERT)                                ▼ (Missing item: SOFT DELETE)
 Update stream URLs, logos, or groups.               Add new channel rows.                               Set deleted_time = UnixEpoch.
 user-overrides remain untouched.                    Binds new category IDs.                             No hard rows delete.
```

### The 4-Step Sync Protocol

1.  **Transactional Enclosure**:
    The entire update executes inside a single database transaction block: `database.withTransaction {}`. If the network drops or the stream is corrupted mid-parse, the transaction is rolled back, preserving the last known good state.
2.  **Incremental Category Diffing**:
    Instead of deleting groups, the sync engine queries existing groups for the active `playlist_id`. It compares incoming category names with the local cache:
    *   *If name matches*: Keeps the existing primary key `id`.
    *   *If new*: Inserts a new row.
    *   *If missing*: Marks the group's `deleted_time = System.currentTimeMillis()`.
3.  **Channel-Level Stream Comparison**:
    Channels are diffed using their unique provider identifier (e.g. the Xtream `stream_id` or the Stalker channel ID, or a hashed combination of M3U stream URL + raw channel name):
    *   **INSERT**: New channels are added.
    *   **UPDATE**: If the stream URL, logo URL, or category association changes, the row is updated. **Crucially, user overrides (favorites, custom names) are preserved.**
    *   **SOFT DELETE**: If a channel is missing from the provider's new playlist import, the app sets its `deleted_time` to the current timestamp. This keeps the row in SQLite, maintaining user configurations if the provider restores the channel later.

---

## 2. PRESERVING USER CUSTOMIZATIONS DURING SYNC

A premium IPTV experience dictates that background playlist refreshes must **never** overwrite custom user modifications. TiViMate guarantees this through separate database columns and junction tables.

### Data Protection Rules

*   **Custom Names Safeguard**:
    When updating a channel row, the parser only overwrites the raw `name` column. The `custom_name` override column is completely bypassed by update transactions, ensuring user custom labels remain active.
*   **EPG Bindings Safeguard**:
    Manual EPG bindings are recorded in the `channel_tvg_bindings` junction table. Since playlist imports only edit the `channels` table, EPG associations survive refreshes without modifications.
*   **Favorites & History Safeguard**:
    The `is_favorite` flag and `last_turn_on_time` values are kept in the channel row. These columns are bypassed by update routines, keeping favorites intact.

---

## 3. PLAYLIST FAILURE SAFETY & FAULT TOLERANCE

If an IPTV server goes down, returning a `503 Service Unavailable`, `404 Not Found`, or a truncated stream, poor players will erase the channel list or freeze.

TiViMate employs a strict **Fault-Tolerant Load System**:
*   **Response Verification**: Before initiating any SQLite writes, the sync engine validates the incoming data. If the playlist parses into **0 channels**, or if the network request fails with an HTTP error, the sync is aborted, and the local database is untouched.
*   **Silent Failures**: Playlist sync failures are handled quietly in the background. The user continues watching their active channels without seeing error dialogues. A small "Update failed" status label is logged in Settings -> Playlists.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Map your playlist sync routines inside a secure `roomDatabase.withTransaction {}` block in Kotlin.
*   Ensure that all update statements explicitly bypass user customization columns (`custom_name`, `is_favorite`).

### Medium-Risk Actions
*   Implement Soft Deletes (`deleted_time`) instead of hard row deletions on all playlist imports.
*   Validate imported stream counts. Abort the transaction if the parsed stream count drops to 0 on a previously populated playlist.

### High-Risk Actions
*   Running massive database syncs on the main UI thread. Always shift the synchronization task to a background thread using `Dispatchers.IO` to prevent scrolling stutters (ANRs) on low-spec hardware.

### What to Measure First
*   Profile EPG sync times. Incremental diffs over 10,000 channels should finish in **under 2 seconds** on standard TV chipsets.

### What Not to Copy Directly
*   Do not copy compiler-obfuscated transaction handlers or transaction callbacks from `C3231.java`.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Resilient incremental updates and user configuration preservation must be implemented to establish playlist stability.
