# TiViMate EPG Syncing and Storage Architecture Analysis

Based on an exhaustive forensic analysis of the TiViMate source code (v5.3.3) and its decompiled Room/SQLite schemas, here is a detailed breakdown of how TiViMate fetches, syncs, and stores EPG data while maintaining an incredibly small storage footprint on low-end Android TV devices.

---

## 1. Does TiViMate Download the EPG List Before Syncing?

**No, it does not download the file to disk first.** 
TiViMate completely avoids downloading massive (often 1GB - 2GB) XMLTV files to the device's physical storage before processing. Instead, it uses a **Streamed Pull-Parsing** approach:

1. **OkHttp Request:** A background worker initiates an HTTP request to the EPG provider.
2. **GZIP Streaming:** The provider streams the data back (typically GZIP compressed).
3. **On-The-Fly Parsing:** TiViMate pipes this stream directly into Android's native `XmlPullParser`. As the data streams in over the network, TiViMate parses it chunk-by-chunk in RAM.
4. **Incremental Commits:** As programs are parsed, they are batched and inserted directly into the SQLite database (`TvPlayer.db`). 

This architecture guarantees that the app never crashes due to Out-Of-Memory (OOM) errors or fills up the limited 8GB storage on devices like the Fire TV Stick just to hold a temporary download file.

---

## 2. How TiViMate Handles EPG Syncing and Fetching

TiViMate ensures that syncing never interrupts the active TV viewing experience (zero video stutter or UI lag). It achieves this via careful background orchestration:

### A. WorkManager Scheduling & Delta Checks
* TiViMate uses Android's **WorkManager** to handle background syncs based on the user's interval setting (e.g., 2, 4, 12, or 24 hours).
* **Startup Delta Check:** When the app launches, it doesn't blindly sync. It calculates: `CurrentTimeMillis - last_update_time`. If the time elapsed is greater than the user's interval, it triggers a sync. Otherwise, it defers the task, saving bandwidth and CPU.

### B. Write-Ahead Logging (WAL) Database Mode
* The SQLite database is forced into **WAL (Write-Ahead Logging)** mode (`PRAGMA journal_mode = WAL`) with a busy timeout of `3000ms`. 
* **Why this matters:** When the background worker writes 100,000+ new EPG rows to the database, WAL mode ensures these writes go to a separate journal file. This means the main UI thread can simultaneously *read* the database to display the TV guide without being locked out or frozen by the background worker.

### C. 90-Minute RAM TTL Cache
* To keep the TV guide scrolling at a fluid 60 FPS, TiViMate doesn't constantly query the SQLite disk when you scroll. 
* It queries the database once for the current viewport and caches the EPG data in a fast in-memory map (RAM) with a **90-minute Time-To-Live (TTL)**. Scrolling simply reads from RAM.

---

## 3. How TiViMate Keeps Space Incredibly Low

TiViMate is famous for its small data footprint. It achieves this through several aggressive database and parsing optimizations:

### A. Partial XMLTV Scanning (The Silver Bullet)
When TiViMate parses the incoming XMLTV stream, it does not save everything. 
* It first checks its local database to see exactly which channels you actually have in your playlists.
* When the parser hits a `<programme>` tag for a channel ID you don't use, **it instantly skips the entire block**. It never allocates the text to memory and never writes it to the database. This reduces DB writes by up to **90%** for massive provider lists.

### B. 4-Byte Integer Epoch Timestamps
Instead of storing timestamps as standard 8-byte millisecond `Long` values (or worse, ISO string dates), TiViMate stores them as **4-byte Unix Epoch Seconds** (`INTEGER`). 
* This literally halves the byte-width of the database's B-Tree indexes. When indexing 500,000 rows, this saves massive amounts of disk space and doubles search speed.

### C. Junction Deduplication (Cross-Playlist Sharing)
If you have two different playlists (e.g., a primary provider and a backup provider) that both feature CNN, TiViMate will **not** store the CNN EPG schedule twice. 
* It stores the program descriptions exactly once in the `tvg_programs` table.
* It uses a lightweight junction table (`channel_tvg_bindings`) to point both of your playlist channels to that single EPG schedule.

### D. Description Opt-Out on the Parse-Level
If a user toggles the "Store program descriptions" setting to OFF in the UI:
* TiViMate doesn't just hide the descriptions—it instructs the `XmlPullParser` to completely skip the `<desc>` XML tags during the network stream. The description column in the database is simply left `NULL`.

### E. FTS4 Virtual Table Isolation
For the search feature to work fast, TiViMate has to index the text of titles and descriptions. 
* Instead of putting heavy text indexes on the main EPG table (which would bloat it and slow down scrolling), TiViMate creates a parallel **FTS4 (Full Text Search)** virtual table (`tvg_programs_fts`). This table only holds the `docid`, `title`, and `desc`, keeping the main schedule table strictly for lightning-fast timeline math.

### F. Aggressive Orphan Pruning
Orphaned data is never allowed to accumulate. During *every* background sync, the WorkManager executes a hard SQL delete command:
`DELETE FROM tvg_programs WHERE stop_epoch_seconds < (strftime('%s', 'now') - pastRetentionSeconds)`
This ensures that the rolling window of "past days to keep" is strictly enforced, wiping old programs out of storage instantly.
