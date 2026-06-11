# TiViMate Backup & Restore System — Technical Deep-Dive

This document provides a highly detailed architectural analysis of **TiViMate’s backup creation and restore restoration engines**. It explains how the application checkpoints active SQLite WAL databases, packages preference files, manages directory selection targets (both local USB/SAF paths and remote SMB shares), verifies compatibility constraints, and executes safe database hot-swaps with automated rollback protection.

---

## 1. THE BACKUP PROCESS (.TMB PACKAGING FLOW)

A TiViMate backup file (`.tmb` extension) is a **compressed ZIP/GZIP archive** containing the application's entire state. To make a backup safe to copy, the app must ensure the database is flushed and consistent.

### The Backup Pipeline

```
 [ Create Backup Click ] ──► [ Open Directory Picker (Local/SMB) ]
                                            │
                                            ▼
                          [ SQLite DB: Force FULL Checkpoint ]
                          * Flush WAL write logs into main DB file
                                            │
                                            ▼
                          [ Package Files in Local Cache ZIP ]
                          - TvPlayer.db (Relational data)
                          - settings.xml (SharedPreferences XML)
                          - backup_metadata.json (Version and MD5 Checksum)
                                            │
                                            ▼
                          [ Pipe Compressed Bytes to Destination ]
```

### Step 1: Database Checkpoint (WAL Flush)
Because TiViMate runs in SQLite **WAL (Write-Ahead Logging)** mode, recent user customizations, history, and playlists are stored in a separate temporary log file (`TvPlayer.db-wal`) rather than the primary database file. 

Before copying the database, the app forces a **Full Checkpoint** to flush all log entries into the primary database file. This ensures that copying only the single `.db` file captures 100% of the active data.
*   *SQL Execution*: `PRAGMA wal_checkpoint(FULL);`

### Step 2: ZIP Compression Packaging
The backup manager creates a temporary ZIP archive in the app's secure internal cache directory (`/data/data/ar.tvplayer.tv/cache/`) and compresses:
1.  **`TvPlayer.db`**: The primary SQLite database.
2.  **`settings.xml`**: A copied snapshot of the private SharedPreferences file.
3.  **`backup_metadata.json`**: A text metadata header:
    ```json
    {
      "app_version_code": 47002,
      "app_version_name": "4.7.0",
      "created_timestamp": 1740000000,
      "md5_checksum": "9cfb3b8c38..."
    }
    ```

### Step 3: Stream Piping
Using a buffered output stream, the compressed bytes are piped directly from the cache to the picked location (local USB, Scoped Storage path, or remote SMB directory using SMBJ client streams), naming the file using a temporal sequence: `tivimate_backup_20260525_211800.tmb`.

---

## 2. THE RESTORE PROCESS (SAFE HOT-SWAP FLOW)

Restoring data is highly risky. Overwriting a database while SQLite connections are open or importing incompatible data will crash the application and corrupt storage directories. TiViMate mitigates this through a **Transactional Hot-Swap Sequence**.

```
 [ Extract Backup to Temp Cache ] ──► [ Parse & Verify backup_metadata.json ]
                                                    │
                                                    ▼ (Verification Checks)
                                      * Version: Backup vCode <= App vCode
                                      * Checksum: Verify file integrity
                                                    │
                                                    ▼ (Close database & active players)
                                      * db.close() & Stop Background Workers
                                                    │
                                                    ▼ (The Hot-Swap)
                                      * Move old DB files to TvPlayer.db.bak (Rollback)
                                      * Copy restored DB and settings.xml to destination
                                                    │
                                      ┌─────────────┴─────────────┐
                                      ▼ (Verify DB Opens)         ▼ (Verification Fails)
                            [ Force App Process Kill ]     [ Restore DB from .bak Rollback ]
```

### The 4 Restoring Safeguards

1.  **Version Compatibility Check**:
    The restorer reads `backup_metadata.json` first. If `backup_metadata.app_version_code > active_app.version_code`, it halts the restore and alerts the user: *"This backup was created with a newer version of the application and cannot be restored."* This prevents database schema mismatch crashes.
2.  **State Cleanup (Closing Connections)**:
    Before swapping files, the app closes all SQLite handles:
    `database.close()`
    It terminates active video playback, shuts down the ExoPlayer pipeline, and stops background WorkManager sync workers.
3.  **Fail-Safe File Swapping**:
    *   **Rollback Copy**: Moves `TvPlayer.db`, `TvPlayer.db-wal`, and `TvPlayer.db-shm` into a temporary rollback directory as `TvPlayer.db.bak`.
    *   **Extraction**: Copies the restored database and XML preference files from the temporary unzip cache to `/data/data/ar.tvplayer.tv/databases/TvPlayer.db` and the corresponding `shared_prefs` folder.
    *   **Validation Check**: Tries to initialize a temporary database connection. If it opens successfully, the rollback `.bak` files are deleted. If it fails (corrupted backup), it immediately restores the `.bak` files, protecting active user data.
4.  **Application Process Kill**:
    Android caches Shared Preferences in memory. Overwriting the XML on disk will not update memory values; the OS will simply overwrite the restored file on the next write. 
    To force the OS to load the restored preference XML and initialize the restored Room DB, the app schedules a **self-restart** via `AlarmManager` and calls:
    `Process.killProcess(Process.myPid())`

---

## 3. HOW YOU CAN APPLY THESE PARADIGMS

### 🟢 What You CAN Use & Adapt:
*   **Force SQLite DB Checkpoint**:
    Before running any database export or backup in your app, flush active WAL logs programmatically:
    ```kotlin
    roomDatabase.openHelper.writableDatabase.execSQL("PRAGMA wal_checkpoint(FULL);")
    ```
*   **Version Guarding**:
    Include a metadata file in your backup zip holding the app's `versionCode`. Block restorations if the backup is from a newer version.
*   **Database Closing & Hot Swaps**:
    Ensure your Room database connection is closed cleanly before copying database files:
    ```kotlin
    roomDatabase.close()
    ```
*   **Verification & Rollback**:
    Copy existing `.db` files to a `.bak` path before replacing them, reverting to the `.bak` files if the restored database fails to open cleanly.

### 🔴 What You CANNOT Copy:
*   **Obfuscated Process Killers**:
    Avoid copying decompiled restart routines from `sources/p262`. Re-implement a standard Android `PendingIntent` restart loop cleanly.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Flush active SQLite WAL log buffers using `wal_checkpoint(FULL)` before exporting backups.
*   Pack the SQLite `.db` file and `shared_prefs` XML file into a standard `.zip` archive.

### Medium-Risk Actions
*   Implement version guarding using JSON metadata sheets in the backup folder.
*   Build an automatic database backup rollback routine that recovers local `.bak` files if restored databases fail to open.

### High-Risk Actions
*   Overwriting active database files while SQLite connections are open. This leads to immediate database corruption and application crashes.

### What to Measure First
*   Benchmark backup creation times. Packaging and compressing a 10MB EPG database should take **less than 800ms** on target hardware.

### What Not to Copy Directly
*   Do not copy custom ZIP utility classes or obfuscated filesystem managers.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Secure, transaction-guarded backup checkpointing, and version-checked hot-swaps.
