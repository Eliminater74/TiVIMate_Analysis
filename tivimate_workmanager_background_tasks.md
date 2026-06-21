# TiViMate WorkManager & Background Task Architecture

This document analyzes how TiViMate handles scheduled, asynchronous, and background operations—primarily automatic EPG syncs and playlist updates—using Android's `WorkManager` framework. Findings are based on the decompiled `C3214.java` and `C1233.java` classes which interface directly with the WorkManager SQLite tables.

---

## 1. THE WORKMANAGER ENGINE

Android `WorkManager` is the recommended OS-level framework for scheduling deferrable, guaranteed background work. TiViMate uses a custom implementation or a shaded/obfuscated version of the WorkManager libraries, as evidenced by its direct manipulation of WorkManager's internal SQLite tables (`workspec`, `dependency`, `workname`, `worktag`).

### 1.1 Why WorkManager?

TiViMate relies on WorkManager because it guarantees execution even if the app is closed or the device reboots. This is critical for IPTV applications:
* **EPG Syncing:** TV guide XMLTV files can be massive (50MB+). They must be downloaded and parsed in the background so the UI doesn't freeze.
* **Playlist Updates:** Xtream Codes and M3U playlists change daily (new VODs added, dead channels removed). Background syncing ensures the user always has fresh data when they launch the app.

---

## 2. BACKGROUND TASK SCHEMA ANALYSIS

The `C3214.java` file acts as the primary Data Access Object (DAO) for the background task engine. It queries the internal `workspec` table.

### 2.1 The `workspec` Table

This table stores the blueprints for every background job. Key columns (from `C3214.java` line 143):

| Column | Description |
|---|---|
| `id` | Unique ID for the work request |
| `state` | Enum: 0=ENQUEUED, 1=RUNNING, 2=SUCCEEDED, 3=FAILED, 4=BLOCKED, 5=CANCELLED |
| `worker_class_name` | The Java class to instantiate to perform the work |
| `initial_delay` | Delay before the job should start |
| `interval_duration` | For periodic jobs (e.g., "Update every 12 hours") |
| `run_attempt_count` | Retry counter |
| `backoff_policy` | Linear or Exponential retry logic on failure |
| `requires_charging` | Constraint: Only run when plugged in |
| `required_network_type` | Constraint: Only run on Wi-Fi/Any network |
| `requires_device_idle` | Constraint: Only run when user isn't actively using the box |

### 2.2 Task Dependencies (`dependency` table)

TiViMate uses WorkManager's capability to chain tasks. For example, it might schedule a task to "Download Playlist," and set a dependent task to "Download EPG" that only runs if the playlist download succeeds.

**Source Evidence** (`C3214.java` line 89):
```sql
SELECT COUNT(*)=0 FROM dependency WHERE work_spec_id=? AND prerequisite_id IN (SELECT id FROM workspec WHERE state!=2)
```
This query checks if a job is clear to run by ensuring all its prerequisites have a state of `2` (SUCCEEDED).

---

## 3. KEY BACKGROUND OPERATIONS

### 3.1 EPG & Playlist Auto-Updates

In the main database `playlists` table (see `tivimate_complete_database_schema_migrations.md`), we see two crucial columns:
* `auto_update` (BOOLEAN)
* `update_interval_hours` (INT)

When a user enables "Auto update EPG" and sets the interval to 12 hours, TiViMate translates this into a `PeriodicWorkRequest` within WorkManager.

The `interval_duration` in the `workspec` table is set to 12 hours (in milliseconds). WorkManager registers this with the Android OS `JobScheduler` or `AlarmManager`. Even if TiViMate is force-closed, the OS wakes the app up exactly every 12 hours, instantiates the `worker_class_name` (e.g., the EPG Sync Worker), and processes the XMLTV file silently.

### 3.2 Network Constraints

Downloading large EPG files on metered mobile data can be costly. TiViMate utilizes the `required_network_type` column (WorkManager's `Constraints.Builder().setRequiredNetworkType(NetworkType.UNMETERED)`).

If the device drops Wi-Fi and switches to cellular, WorkManager immediately pauses the background sync and holds it in the `ENQUEUED` state until unmetered Wi-Fi is restored.

### 3.3 Retry and Backoff

If an IPTV portal is down (HTTP 502/504), the worker returns a `Result.retry()`.
WorkManager references the `backoff_policy` and `backoff_delay_duration` columns. If exponential backoff is set, it will retry in 10 minutes, then 20, then 40, preventing the app from DDoS-ing the provider's server.

### 3.4 Cleanup Mechanisms

Over time, completed jobs clutter the database. TiViMate actively cleans up stale task records.

**Source Evidence** (`C1233.java` line 561):
```sql
DELETE FROM workspec WHERE state IN (2, 3, 5) AND (last_enqueue_time + minimum_retention_duration) < [System.currentTimeMillis() - threshold]
```
This query safely purges SUCCEEDED (2), FAILED (3), and CANCELLED (5) jobs that are older than a specific retention threshold, keeping the `workspec` table lean and fast.
