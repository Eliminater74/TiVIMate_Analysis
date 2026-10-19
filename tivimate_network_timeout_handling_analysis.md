# TiViMate Network Timeout & Fault Tolerance Architecture

This document provides a comprehensive technical breakdown of how **TiViMate (v5.3.3)** handles network timeouts, server disconnects, hung sockets, and background sync failures across all layers of its application architecture.

---

## 1. Overview: The "Fast-Fail & Silent Recovery" Philosophy

In IPTV applications on low-end TV hardware (e.g., Fire TV Sticks, Android TV boxes), default OS network timeouts (often 30–60 seconds) cause disastrous user experiences: black screens, frozen UI threads, app crashes, and corrupted databases.

TiViMate strictly adheres to a **Fast-Fail & Silent Recovery** model:
1. **Aggressive Low Timeouts (5s)** to fail fast instead of hanging.
2. **Atomic Rollbacks** to protect local database integrity when network drops occur mid-stream.
3. **Exponential Backoff & Watchdogs** to automatically recover streams or background tasks without user intervention.

---

## 2. Layer 1: HTTP & Socket Timeout Configuration (OkHttp)

TiViMate bypasses standard Java `HttpURLConnection` and routes all network operations (Playlist downloads, EPG XMLTV streams, Xtream Codes API requests, and ExoPlayer video segment fetches) through a tuned **OkHttp Client Pool**:

* **Connect Timeout:** `5,000 ms` (5 Seconds)
* **Read Timeout:** `5,000 ms` (5 Seconds)
* **Write Timeout:** `5,000 ms` (5 Seconds)
* **Connection Pooling:** TCP/TLS sockets are retained and reused across requests. If a provider's server is alive, socket reuse yields `0ms` handshake latency.

### Why 5-Second Timeouts?
If an IPTV server or EPG host stops responding, a standard 30-second timeout leaves the viewer stuck on a spinning wheel. By capping timeouts at 5 seconds, TiViMate immediately detects dead sockets, aborts hung requests, and triggers retry or fallback logic instantly.

---

## 3. Layer 2: Background EPG & Playlist Sync Timeout Handling

When TiViMate updates a playlist or fetches an XMLTV EPG list in the background, a network timeout can happen at any moment (e.g., halfway through streaming a 500MB XML file).

Here is how TiViMate handles it:

### A. Atomic Transaction Rollbacks (`database.withTransaction {}`)
* All playlist and EPG updates are executed inside a single, atomic SQLite database transaction.
* **The Safety Mechanism:** If a network socket times out mid-transfer, an `IOException` or `SocketTimeoutException` is thrown. The transaction immediately **rolls back entirely**.
* **Result:** The local SQLite database (`TvPlayer.db`) is left completely untouched in its last known good state. **Existing channels, EPG schedules, and user configurations are never wiped or corrupted by a timed-out update.**

### B. The Zero-Data Guard
Before committing imported stream channels or EPG programs to disk, TiViMate validates the parsed payload count.
* If a network timeout or dropped connection results in **0 channels** or 0 EPG entries, the transaction aborts automatically. This prevents accidental deletion of existing channels due to server errors.

### C. WorkManager Retries & Exponential Backoff
Background sync jobs are managed by Android's **WorkManager**:
* When a network timeout occurs during a background sync, WorkManager catches the failure and applies an **Exponential Backoff Policy** (e.g., retrying after 1s → 2s → 4s, scaling up to minutes/hours).
* Sync failures run quietly in the background—they **never** display intrusive popup dialogs over active playback. A discrete status label ("Update failed") is logged in `Settings ➔ Playlists`.

---

## 4. Layer 3: Live Stream Playback & Buffering Watchdogs

When streaming live video, network congestion or server drops can cause socket stalls where no data arrives, but the connection hasn't formally closed.

### A. The 5-Second Buffering Watchdog
1. When ExoPlayer enters `STATE_BUFFERING`, TiViMate registers a scheduled watchdog timer capped at **5,000 ms**.
2. If network packets resume within 5 seconds, the watchdog is canceled and playback continues.
3. If 5 seconds elapse without data (socket hung), the watchdog fires a synthetic timeout exception to force recovery.

### B. The 3-Step Auto-Retry Engine
When a playback stream times out or throws an HTTP error (403, 404, 503, `SocketTimeoutException`):
1. **Immediate Retry (Step 1):** Retries the active stream URL up to 3 times with exponential delays (1s → 2s → 4s).
2. **Session Re-Handshake (Step 2):** If retries fail, it assumes the session or token timed out. It re-executes authentication (e.g., refreshing Stalker portal MAC tokens or Xtream Codes session keys) and requests a fresh stream URL.
3. **Software Decoder / Alternative Fallback (Step 3):** If connection failures persist, it falls back to an alternative stream URL (if available in the playlist) or displays a clean, user-friendly error overlay ("An error occurred: Code 403. Retrying...").

---

## 5. Layer 4: Database Concurrency & Busy Timeouts

Network updates write tens of thousands of rows in the background while the user reads EPG data on the UI thread. Network packet bursts can cause database write spikes.

* **SQLite Busy Timeout:** Configured via `PRAGMA busy_timeout = 3000;` alongside `PRAGMA journal_mode = WAL;`.
* **Behavior:** If a network parse commit collides with a UI read query, SQLite will wait up to **3,000 ms** for the lock to clear rather than throwing an immediate `SQLiteDatabaseLockedException` or crashing the app.

---

## 6. Summary Matrix: Timeout Handling by Scenario

| Scenario | Primary Mechanism | Timeout Threshold | Recovery Action |
| :--- | :--- | :--- | :--- |
| **EPG XML Download Timeout** | OkHttp Socket Timeout | 5 Seconds | WorkManager Exponential Backoff + DB Transaction Rollback |
| **Playlist Stream Drop Mid-Parse** | `database.withTransaction` | Immediate on `IOException` | Atomic Rollback (Local data untouched) |
| **Live Stream Buffer Freeze** | Player Watchdog Handler | 5 Seconds | Buffering Watchdog triggers 3-Step Auto-Retry |
| **Stalker / XC Token Timeout** | HTTP 403 / Socket Expiration | 5 Seconds | Silent Auth Handshake + Socket Re-connection |
| **Database Write Collision** | SQLite `busy_timeout` | 3,000 ms | SQLite queues lock for 3s before throwing exception |

---

## Key Takeaway for Developers

TiViMate's resilience comes down to **never trusting the network**:
1. Keep socket timeouts short (**5 seconds**).
2. Wrap all network-driven database imports in atomic transactions (`withTransaction {}`) so timeouts roll back changes safely.
3. Handle sync errors quietly in the background without interrupting the active video playback experience.
