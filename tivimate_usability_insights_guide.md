# TiViMate Architectural Audit — Usability Insights Guide (What You Can vs. Can't Use)

This master guide summarizes the technical audit of the decompiled **TiViMate** source code. Its purpose is to clearly separate the clean, high-performance architectural patterns, database queries, and libraries that you **can** adapt for your app from the compiler-generated artifacts, obfuscated entities, and R8 layout bugs that you **cannot** (and should not) copy.

---

## 1. SQL DATABASE & SCHEMAS (`C3210.java`, `C3214.java`, `C3220.java`)

SQLite query strings are stored as plain text literals in the dex files. Because of this, the compiler cannot obfuscate them. They represent a goldmine of optimized IPTV relational database structures.

### 🟢 What You CAN Use & Adapt:
* **The WAL Pre-Warming Pattern**:
  Using `PRAGMA journal_mode = WAL;`, `PRAGMA synchronous = NORMAL;`, and `PRAGMA busy_timeout = 3000;`. This is fully safe and highly recommended to prevent database locks when background workers write massive EPG updates.
* **Epoch Seconds for Timestamps**:
  Storing EPG schedules and playback markers as Unix Epoch Seconds (`INTEGER` in SQLite) instead of millisecond `Long` values. This halves index byte sizes and dramatically speeds up binary-seek sorting.
* **FTS4 Virtual Tables**:
  Creating parallel virtual search tables (`tvg_programs_fts`, `movies_fts`, `series_fts`) and matching queries via `MATCH` syntax:
  ```sql
  SELECT * FROM tvg_programs
  INNER JOIN tvg_programs_fts ON tvg_programs.id == tvg_programs_fts.docid
  WHERE tvg_programs_fts.title MATCH ? AND start_epoch_seconds >= ?
  ```
* **Aggregate Group-by Joins**:
  Running aggregate joins linking playlists and VOD categories dynamically to pre-compute movie counts and maximum added dates in a single SQL execution.

### 🔴 What You CANNOT Copy:
* **Room-Generated DAO Interfaces**:
  Decompiled classes like `C3210.java` or `C3214.java` do not contain human-written Room structures. They are the **raw Java implementations** compiled by the Room processor at build time. Copying them directly will cause immediate compilation failure.
* **Obfuscated Table & Column Mappings**:
  Avoid hardcoded index lookups such as:
  ```java
  int iM75427 = AbstractC3586.m7546(interfaceC4930, "catchup_type");
  ```
  Instead, leverage modern Room annotations (`@Entity`, `@Dao`, `@Query`, `@ColumnInfo`) in clean Kotlin and let the compiler generate these binding bridges.

---

## 2. THE 전자 PROGRAM GUIDE (EPG) GRID RENDERER

TiViMate's liquid-smooth 60fps scrolling is achieved by entirely bypassing standard Android view groups.

### 🟢 What You CAN Use & Adapt:
* **Direct Canvas Pixel Layout**:
  Bypassing nested vertical/horizontal `RecyclerView` adapters. Replicating a custom `View` that overrides `onDraw(canvas: Canvas)` and draws program boxes as simple geometry (`canvas.drawRect()`).
* **Binary Search Viewport Queries**:
  In-memory sorting of EPG programs on startup. When rendering or scrolling, use binary search to quickly locate only the program indexes that intersect the active viewport start and end times:
  $$startX = (programStartSeconds - viewportStartSeconds) \times pixelsPerSecond$$
* **Multi-Tiered Cache Allocations**:
  Implementing a strict 90-minute RAM cache to check for EPG updates, decoupled entirely from active UI rendering.

### 🔴 What You CANNOT Copy:
* **Obfuscated Custom Graphics Classes**:
  Do not copy any direct drawing classes or bitmap processors from packages like `p179` or `p262`. The variable assignments, layouts, and canvas drawing indices have been optimized for mathematical efficiency by R8, making them completely unreadable and inflexible.
* **Obfuscated Grid Adapters**:
  Avoid attempting to copy Leanback base adapters or layout managers that reference obfuscated grid layouts.

---

## 3. PLAYLIST PARSING & STREAM LOADING (`C1367.java`)

Importing large M3U files (10MB+) without causing Out-of-Memory (OOM) crashes on 1.5GB RAM Android TV boxes.

### 🟢 What You CAN Use & Adapt:
* **Streaming Line-by-Line Tokenization**:
  Wrapping the incoming URI or file stream in a `BufferedReader` over an `InputStreamReader`. Read characters sequentially rather than downloading the entire playlist file as a single String in RAM.
* **Single-Pass Attribute Parser**:
  Instead of compiling heavy Java `Pattern` regex matchers for every line, write a custom character-skipping scanner that seeks string segments (`tvg-id=`, `tvg-logo=`, `group-title=`) in a single parsing loop.
* **Batch Room Transactions**:
  Gathering parsed channels into lists of 500 or 1,000 items and committing them within a single transaction `database.withTransaction {}` to maximize SQLite disk write speeds.

### 🔴 What You CANNOT Copy:
* **Internal ExoPlayer Media Builders**:
  Avoid copying the decompiled parsing bridges that hook into ExoPlayer's private format descriptors (e.g. `C1490` or `C1367` models). These change frequently between ExoPlayer/Media3 releases and will break on library updates.

---

## 4. MULTIVIEW (MULTI MODE) ENGINE

Displaying up to 9 channels concurrently without melting standard Android TV processors.

### 🟢 What You CAN Use & Adapt:
* **Muted Background Decoders**:
  Ensuring only the actively highlighted player has audio focus. Call `.setVolume(0f)` on all secondary player instances to disable audio decoding and save significant CPU overhead.
* **Decoder Limit Exception Handling**:
  Catching the standard `ExoPlaybackException` with code `DECODER_INIT_FAILED`. Instead of crashing, gracefully catch the error and notify the user that their device's hardware limits have been reached, or fallback automatically to software rendering decoders like FFmpeg extensions.

### 🔴 What You CANNOT Copy:
* **Custom ViewGroup Focus Hooks**:
  Do not copy the decompiled D-pad layout managers designed to swap focus between grid views. R8 removes field names, making it impossible to debug view hierarchies or add new player configurations. Write a clean Kotlin `GridLayout` or `ConstraintLayout` wrapper to handle layout positioning and focus redirection.

---

## 5. NETWORK BROWSER & SAMBA / SMB ACCESS (`p078.C1671`)

Integrating network shares for backup, restore, and PVR recording storage.

### 🟢 What You CAN Use & Adapt:
* **SMBJ Client Library**:
  Use `com.hierynomus.smbj` for SMBv2/v3 support. Avoid the outdated, insecure, and slow `jcifs` library which is locked to SMBv1.
* **Polymorphic File Model**:
  Replicating a clean, unified data model that represents both local files (`java.io.File`) and remote network files (SMB streams) behind a shared virtual directory abstraction.

### 🔴 What You CANNOT Copy:
* **`p078` Helper Classes**:
  Do not copy class files such as `C1671.java` or `AbstractC1679.java`. They contain obfuscated connection-pooling handles and network retry-policy fields that have been compiled into flat, hard-coded integer indexes. Re-implement your SMB connections using clean Kotlin coroutine wrappers.

---

## 6. PVR & SCHEDULER SUB-SYSTEM (`RecordingForPush`, `RecordingRepeatRules`)

Persisted, resilient scheduled recording that survives reboots and TV CEC sleep states.

### 🟢 What You CAN Use & Adapt:
* **Foreground Service with WakeLocks**:
  Registering your recording service as a persistent foreground process holding a `PARTIAL_WAKE_LOCK`. This ensures the TV box's CPU stays awake to pull stream data even when the display panel is shut off.
* **Direct Stream Piping (TS File Dump)**:
  Bypassing active video rendering during recording. Direct the raw HTTP input stream socket to a `BufferedOutputStream` writing straight to a `.ts` file, avoiding expensive video decoding loops.
* **Stream Watchdog and Reconnection**:
  Implementing a background watchdog that monitors write progress. If a stream times out, trigger an immediate reconnection loop and append bytes to the same file.

### 🔴 What You CANNOT Copy:
* **`RecordingForPush` and IPC Wrappers**:
  Do not copy the decompiled models in `ar.tvplayer.core.data.db.entities`. They represent highly specific obfuscated serialization layouts designed to bind with custom alarm receivers. Build a native, clean Room entity representing your recording tasks.

---

## 7. CATCH-UP / TIMESHIFT RESOLUTION & STALKER PORTALS

### 🟢 What You CAN Use & Adapt:
* **Dynamic Time Interpolation**:
  Rewriting timeshift stream URLs dynamically based on catchup attributes (`catchup_type`, `catchup_source`).
* **Stalker Session Broker Architecture**:
  Storing active portal session tokens (`stalker_token`) in SQLite. If a stream fails with a HTTP 403 or network socket error during playback, catch the error, trigger a background handshake `/portal.php?type=itv&action=create_link`, update the database token, and swap the player URL dynamically.

### 🔴 What You CANNOT Copy:
* **MAG/Stalker Network Handshakers**:
  Avoid copying decompiled Stalker network wrappers. Stalker portals use unstable, vendor-specific API formats. Write clean, type-safe API clients using Retrofit or Ktor and encapsulate authentication failures inside clean Repository layers.

---

## SUMMARY OF THE CORE CRITICAL WARNINGS

| Subsystem | What You CAN Replicate | What You MUST NOT Copy |
|---|---|---|
| **Database** | SQL strings, WAL parameters, FTS4 queries, integer epoch times | Room-generated Java source files (`C3214`, `C3220`), obfuscated index mappings |
| **UI Guide** | Custom Canvas pixel calculations, binary search viewport queries | R8-obfuscated drawing classes, Leanback internal layout modifications |
| **Playlist** | BufferedReader streaming input, character-skipping tokenizers | Hardcoded format index models, ExoPlayer private binder integrations |
| **Multiview** | Muted background decoders, hardware limit exception recovery | Decompiled layout coordinate calculators, custom focus grid wrappers |
| **SMB Share** | SMBJ library integration, polymorphic local/remote models | Obfuscated connection socket loops, obfuscated retry-policy classes |
| **PVR / DVR** | Foreground Service types, CPU WakeLocks, raw TS file writing | Decompiled alarm models, serialized type-converter helpers (`C0185`) |
| **Catch-up** | Dynamic token interpolation, Stalker API auto-refresh flows | Obfuscated portal handshakers, magical status query integers |
