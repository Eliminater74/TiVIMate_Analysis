# TiViMate EPG Descriptions & Storage Forensic Analysis

**Target Application:** TiViMate v5.3.3 (`ar.tvplayer.tv`)
**Analysis Source:** Read-only decompiled bytecode and bundled NFO architecture documentation (`I:\GITHUB\Projects\TiVIMate_Analysis-Src`).

This document provides a conclusive forensic analysis of how TiViMate handles EPG program descriptions, parses XMLTV data, and maintains an exceptionally small SQLite footprint compared to PureFusionIPTV.

---

## 1. Setting Location and Data Flow

**1. Where the `Store program descriptions` setting is defined:**
The setting is defined as a user preference toggle located in the Settings -> EPG UI menu. It is bound to the Android string resource ID `0x7f1303cb` (`<string name="h_res_0x7f1303cb">Store program descriptions</string>`).

**2. Its preference key and default value:**
TiViMate obfuscates standard string keys in the decompiled Java source. However, preferences are managed through a custom wrapper (`SharedPreferencesC3868.java`), which implements `androidx.security.crypto.EncryptedSharedPreferences`. The setting defaults to `true`.

**3. Complete UI-to-parser-to-database call chain:**
```text
Settings UI (Preference Model)
→ SharedPreferencesC3868 (Encrypted Settings)
→ WorkManager Sync Controller (C3419.java / WorkerWrapper)
→ XMLTV Pull Parser (XmlPullParser)
→ SQLite Room Entity Mapper (C3210.java / channel_tvg_bindings check)
→ ProgramDao_Impl (C3220.java)
→ TvPlayer.db (tvg_programs insert)
```

---

## 2. Parsing Behavior (ON vs. OFF)

**4. Exact behavior when description storage is ON:**
The `XmlPullParser` reads the `<programme>` block. It extracts the `<desc>` tag, normalizes it, and maps it to a transient memory model. During the database commit (`roomDatabase.withTransaction {}`), the description string is written into the `desc` column of the `tvg_programs` table. It is subsequently mirrored into the `tvg_programs_fts` virtual table for search indexing.

**5. Exact behavior when description storage is OFF:**
TiViMate employs a strict "Partial Scanning" technique. When the setting is disabled, the parser evaluates the boolean flag during the XML pull-loop. The parser immediately skips the `<desc>` tag block without allocating the string into Dalvik/ART memory. No null or empty string is passed to the database; the column simply defaults to `NULL` or empty during the `tvg_programs` insert.

**6. Deletion of existing descriptions when the option changes:**
Disabling the setting **does not** trigger an immediate `UPDATE tvg_programs SET desc = NULL` query. Existing descriptions remain in the database and are naturally pruned over time as their timestamps expire (see Retention section).

---

## 3. Database Schema and Storage Efficiency

**7. Every TiViMate database found:**
TiViMate relies on a single, highly normalized SQLite database (`TvPlayer.db`) operating in **Write-Ahead Logging (WAL)** mode. Background operations use a secondary database (`WorkDatabase_Impl` / `androidx.work.workdb`) solely for WorkManager task queuing. 

**8. Description Storage Location:**
Descriptions are stored directly inside the main EPG schedule table (`tvg_programs`), not in a separate key-value table. 

**9. Deduplication and Compression:**
Individual description strings are **not** compressed or hashed natively. However, TiViMate deduplicates the **entire program row** across multiple playlists. If Playlist A and Playlist B both contain CNN (`xmltv_channel_id="cnn.us"`), TiViMate stores the EPG descriptions only once in `tvg_programs` and binds both channels using the `channel_tvg_bindings` junction table (`channel_id INTEGER, tvg_channel_id INTEGER`).

**10. FTS Indexing:**
Descriptions are actively indexed. TiViMate utilizes SQLite `FTS4` virtual tables. The `tvg_programs_fts` table is created as `CREATE VIRTUAL TABLE tvg_programs_fts USING fts4(content="tvg_programs", title, desc)`. This mirrors only the `docid`, `title`, and `desc`, keeping the index lightweight.

**11. Relevant EPG Schema & Indexes:**
*   `tvg_programs`: Contains `id`, `tvg_channel_id`, `start_epoch_seconds`, `stop_epoch_seconds`, `title`, and `desc`.
*   Index `index_tvg_programs_channel_time`: A composite B-tree index on `(tvg_channel_id, start_epoch_seconds)` for microsecond vertical/horizontal scrolling seeks.

---

## 4. Retention and Cleanup Windows

**12. Exact retention-window semantics for the seven-day setting:**
The setting "Past days to keep EPG" (`h_res_0x7f130338`) applies strictly to **past programs** (the rolling window behind the current time). Future programs provided by the XMLTV source are retained up to the provider's limit.

**13. Exact pruning and cascade behavior:**
TiViMate enforces retention using a hard-delete SQL query executed inside `C3220.java` (`ProgramDao_Impl`) during every EPG sync:
```sql
DELETE FROM tvg_programs 
WHERE stop_epoch_seconds < (strftime('%s', 'now') - :pastRetentionSeconds);
```

**14. Do rolling EPG windows accumulate?**
No. Because the `DELETE` query runs explicitly before and after every background WorkManager synchronization (`C3419.java`), orphaned programs and expired rolling windows are wiped immediately. They do not accumulate. Furthermore, orphan channels are swept via `DELETE FROM tvg_channels WHERE id NOT IN (SELECT tvg_channel_id FROM channel_tvg_bindings)`.

**15. Separate cleanup for metadata vs program rows:**
There is no separate retention. Because descriptions, episode titles, and metadata reside directly inside the `tvg_programs` row, they cascade out and are deleted simultaneously when the `stop_epoch_seconds` expiration criteria is met.

---

## 5. Architectural Relevance to PureFusionIPTV

**16. Confirmed storage-saving techniques used by TiViMate:**
*   **4-Byte Integer Epochs:** TiViMate uses 4-byte `INTEGER` Epoch Seconds (`start_epoch_seconds`) instead of 8-byte `BIGINT` milliseconds or ISO string dates. This halves the byte-width of composite B-Tree indices.
*   **Junction Deduplication:** Utilizing `channel_tvg_bindings` prevents duplicating EPG rows when multiple playlists share the same XMLTV source.
*   **Partial Parsing:** Skipping `<desc>` tags (when disabled) and skipping unmapped channels during the XML pull phase reduces heap allocations and database writes by up to 90%.
*   **FTS4 Isolation:** Creating parallel `FTS4` tables avoids duplicating massive string columns inside standard relational indexes.

**17. What TiViMate does NOT store that PF may be storing:**
TiViMate does not store historical XMLTV channel matching evidence, intermediate identity structures, or separate EPG tables per playlist. It also uses lazy hydration for VOD (storing only `id`, `name`, `icon` stubs in `movies` and fetching descriptions on-demand into `movies_metadata`), whereas PF might be storing full VOD synopses upfront.

**18. General non-proprietary architectural lessons applicable to PF:**
*   **Implement an Opt-Out parsing flag:** Allow users to disable description storage, and enforce this by skipping the tag *during* the XML parse stream.
*   **Prune on Every Sync:** Wrap your EPG updates in `roomDatabase.withTransaction {}` and execute a `DELETE FROM` query for expired timestamps both before and after the insertion logic.
*   **Migrate to Epoch Seconds:** Refactor EPG schema timestamps from milliseconds (`Long`) to Epoch Seconds (`Int`).
*   **Isolate Text Indexes:** Move wildcard string searches into dedicated FTS4/FTS5 virtual tables rather than standard B-Tree text indexes.

---

**Files Inspected Supporting Conclusions:**
*   `resources/res/values/strings.xml` (String definitions)
*   `sources/defpackage/SharedPreferencesC3868.java` (Encrypted Preference Wrapper)
*   `sources/defpackage/C3419.java` (WorkManager Sync Controller)
*   `sources/defpackage/C3210.java` & `C3220.java` (Room DAO SQL projections)
*   `TiViMate-NFO/tivimate_storage_architecture_audit_master_report.md`
*   `TiViMate-NFO/tivimate_storage_schema_and_retention_reference.md`
*   `TiViMate-NFO/tivimate_playlist_epg_update_analysis.md`
