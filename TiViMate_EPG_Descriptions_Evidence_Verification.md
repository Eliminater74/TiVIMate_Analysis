# TiViMate EPG Descriptions & Storage: Evidence Verification Report

**Target Application:** TiViMate v5.3.3 (`ar.tvplayer.tv`)
**Analysis Source:** Read-only decompiled bytecode (`I:\GITHUB\Projects\TiVIMate_Analysis-Src`)

## 1. Executive Verdict

The previous forensic analysis report relied heavily on existing `TiViMate-NFO` documents which appear to have been generated from dynamic analysis, SQLite dumping, or a previous un-obfuscated version of the application. 

**Static Analysis Verdict:** The current decompiled Java bytecode in this repository is heavily protected by a string encryptor or obfuscator. Crucial SQL schema definitions, database queries (`INSERT`, `DELETE`), preference keys, and XMLTV tag literals (`<programme>`, `<desc>`) have been entirely scrubbed or encrypted into byte arrays/methods. Therefore, while the application's UI strings (via `strings.xml`) confirm the existence of these features, the exact internal persistence mechanics (such as `tvg_programs`, FTS tables, and integer epochs) **cannot be proven** via direct source code evidence in this repository and must be classified as architectural **INFERENCES**.

---

## 2. Claim-by-Claim Verification Table

| Claim | Status | Evidence Type Found |
| :--- | :--- | :--- |
| `Store program descriptions` setting exists | **PARTIALLY CONFIRMED** | `strings.xml` resource |
| Description parsing is skipped when disabled | **UNPROVEN** | Obfuscated parser logic |
| Descriptions stored in `tvg_programs` | **UNPROVEN** | Obfuscated SQL schemas |
| TiViMate uses `tvg_programs_fts` | **UNPROVEN** | Obfuscated SQL schemas |
| Programs shared via `channel_tvg_bindings` | **UNPROVEN** | Obfuscated SQL schemas |
| Old programs pruned on every EPG sync | **UNPROVEN** | Obfuscated DAO calls |
| TiViMate avoids history identity tables | **UNPROVEN** | Obfuscated SQL schemas |

---

## 3. Setting Evidence

**Status:** `PARTIALLY CONFIRMED`
**Evidence:** 
*   **Visible Label:** `<string name="h_res_0x7f1303cb">Store program descriptions</string>` found in `resources/res/values/strings.xml:752`.
*   **Resource ID:** `0x7f1303cb`
**Conclusion:** The setting label explicitly exists. 
**Confidence:** Low for internal mechanics. The stored preference key, default value, and persistence mechanism (e.g., `SharedPreferences`) cannot be directly verified because the string ID `0x7f1303cb` (and its decimal equivalent) is stripped or encrypted in the compiled Java bytecode (`sources/defpackage/*.java`).

## 4. Parser Call Chain

**Status:** `UNPROVEN` (Blocked by Obfuscation)
**Evidence:** 
*   While `XmlPullParser` is utilized natively in the app (e.g., MPEG-DASH parsing in `C3196.java`), no cleartext references to XMLTV tags (`<programme>`, `<title>`, `<desc>`) exist anywhere in the decompiled Java source. 
**Conclusion:** Cannot trace the setting from UI to XMLTV parsing logic. The claim that `<desc>` is skipped during parsing remains an INFERENCE. 

## 5. Database and Schema Evidence

**Status:** `UNPROVEN` (Blocked by Obfuscation)
**Evidence:** 
*   **WorkManager Schemas Only:** Cleartext SQL statements (`CREATE TABLE`, `INSERT INTO`) exist *only* for the `androidx.work` background worker database (e.g., `_new_WorkSpec` in `C3764.java` and `C3787.java`). 
*   **Room Engine Confirmed:** A Room database validation helper (`room_master_table`) exists in `C4115.java`, proving the Room ORM is utilized.
*   **EPG Schema Missing:** Zero cleartext references to `tvg_programs`, `channel_tvg_bindings`, or `my_programs` exist in the bytecode.
**Conclusion:** The exact EPG database schema cannot be directly verified. Claims about specific table names and database structures are INFERENCES derived from external or dynamic analysis.

## 6. Description Persistence Behavior

**Status:** `UNPROVEN`
**Conclusion:** Due to the absence of cleartext DAO queries or `INSERT` statements for EPG data, it cannot be conclusively determined if descriptions are stored in the primary program row, a separate table, or dropped entirely. 

## 7. FTS Behavior

**Status:** `UNPROVEN`
**Evidence:** No `CREATE VIRTUAL TABLE`, `FTS3`, `FTS4`, or `MATCH` query strings are present in the decompiled Java source. 
**Conclusion:** The existence of a dedicated search table (`tvg_programs_fts`) remains an INFERENCE.

## 8. Retention and Pruning

**Status:** `PARTIALLY CONFIRMED` (UI Only)
**Evidence:** 
*   **Visible Label:** `<string name="h_res_0x7f130338">Past days to keep EPG</string>` is present in `strings.xml:605`.
**Conclusion:** The setting UI label exists. However, the exact delete query, timestamp comparison (`stop_epoch_seconds < ...`), and WorkManager execution timing cannot be verified due to obfuscation of the DAO classes.

## 9. Cross-Playlist/Source Deduplication

**Status:** `UNPROVEN`
**Conclusion:** The relational junction table `channel_tvg_bindings` does not appear in the decompiled source. The claim that multiple playlists share one EPG schedule remains an INFERENCE.

## 10. Timestamp Representation

**Status:** `UNPROVEN`
**Conclusion:** Cannot verify whether timestamps use 4-byte `INTEGER` Epoch Seconds or 8-byte `BIGINT` due to the lack of cleartext SQLite schema definitions in the repository.

## 11. Description Toggle Lifecycle

**Status:** `UNPROVEN`
**Conclusion:** Without access to the preference key listener or parser logic, the exact lifecycle (immediate database compaction vs. natural expiration) cannot be mapped via static analysis.

---

## 12. Confirmed Storage-Saving Architecture

Only the following UI-level features are **CONFIRMED**:
*   A user toggle to disable EPG descriptions.
*   A configurable retention window ("Past days to keep EPG").

All other architectural claims (Description opt-out parsing logic, Shared schedules across playlists, Compact epoch timestamps, FTS design, Orphan cleanup) remain **INFERENCE**.

## 13. Unproven or Corrected Claims

*   **Corrected:** The previous report asserted explicit table names (`tvg_programs`, `movies_metadata`) and exact SQL queries as if they were visible in the Java bytecode (e.g., `C3220.java`). This pass confirms that the current decompilation in `defpackage/` **does not** contain these cleartext strings. The previous analysis was likely derived from SQLite DB dumping or an older, unobfuscated APK.

## 14. Non-Proprietary Lessons for PureFusionIPTV

Before modifying PureFusionIPTV based on inferred TiViMate designs, PF should measure its own baseline:
1.  **Description Footprint:** Quantify the exact byte overhead of descriptions in the primary EPG rows vs. FTS tables.
2.  **Duplication Metrics:** Measure how many duplicate XMLTV records exist across multiple playlists. If high, a junction table (like the inferred `channel_tvg_bindings`) is a mathematically sound architectural upgrade regardless of TiViMate's implementation.
3.  **Opt-Out Parser:** Implementing an XML pull-parser check to skip `<desc>` blocks entirely when a "lite EPG" setting is enabled is a universal best practice for memory-constrained devices.
4.  **Index Width:** Evaluate if moving from `BIGINT` milliseconds to `INT` epoch seconds materially reduces B-Tree index size on disk.

## 15. Material Files Inspected

*   `resources/res/values/strings.xml`
*   `sources/defpackage/C3196.java` (XmlPullParser usage)
*   `sources/defpackage/C3764.java` / `C3787.java` (WorkManager SQL Schemas)
*   `sources/defpackage/C4115.java` (Room Database Validation)
