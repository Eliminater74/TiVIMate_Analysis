# TiViMate Playlist Setup Wizard & Tuning Pipeline — Technical Deep Dive

This document details the step-by-step technical execution of TiViMate's **Playlist Setup Wizard**, tracing the system from the initial UI interaction, through multi-protocol validation handshakes, SQLite Room insertions, and background database syncing, all the way to ExoPlayer buffering and live video rendering.

---

## The End-to-End Wizard Lifecycle at a Glance

The playlist setup and playback loop spans UI inputs, OkHttp networks, SQLite transactional writes, and media decoding:

```
[ User Initiates "Add Playlist" ]
              │
              ▼
[ Select Playlist Type (M3U / XC / Stalker) ]
              │
              ▼
[ Enter Credentials / URLs (Emulated Forms) ]
              │
              ▼
[ Background Network Validation & Verification ]
  * M3U: Streaming Line Scan (Checks #EXTM3U)
  * XC: /player_api.php?username=... Handshake
  * Stalker: /portal.php?action=handshake (MAC address)
              │
              ▼
[ EPG Source Setup & Playlist Naming ]
              │
              ▼
[ Atomic Room Database Commit (WAL Transactions) ]
              │
              ▼
[ Ingestion & Background Sync (WorkManager) ]
              │
              ▼
[ Load Playback Viewport & Zap first channel ]
              │
              ▼
[ Dynamic URL Resolution & ExoPlayer Buffering ]
              │
              ▼
[ SurfaceView Overlay Render -> TV Watching! ]
```

---

## 1. Wizard Initiation & Type Selection

When a user opens TiViMate for the first time or navigates to **Settings ➔ Playlists**, they are greeted with an empty state prompting them:
> *"To watch TV channels, please add a playlist provided by your IPTV service"* (String: `u_res_0x7f130427` in [strings.xml](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/resources/res/values/strings.xml)).

Clicking **Add playlist** (`u_res_0x7f13025c`) launches the Setup Wizard, built using a customized leanback-styled guided step fragment interface.

### The Playlist Types

The user is prompted to select a **Playlist type** (`u_res_0x7f130341`), backed by three globally translated string identifiers:
1. **M3U playlist** (`u_res_0x7f13031d`): Standard M3U/M3U8 text files.
2. **Xtream Codes** (`u_res_0x7f1303f1`): Server-side API billing panels.
3. **Stalker Portal** (`u_res_0x7f1303b5`): MAG-emulated portal servers.

---

## 2. Input Parameters & Forms

Depending on the selection, the wizard dynamically inflates text boxes and remote-friendly inputs:

### Form Configurations by Protocol

| Playlist Type | User Input Required | Database Mapping in `playlists` | Resource String References |
|---|---|---|---|
| **M3U Playlist** | Playlist URL or Local File Path | `url` | `u_res_0x7f130343` (Playlist URL)<br>`u_res_0x7f130310` (Select local file) |
| **Xtream Codes** | Server URL, Username, Password | `url` (Server base), `username`*, `password`* (*XC credentials) | `u_res_0x7f130073` (Username)<br>`u_res_0x7f13006c` (Password) |
| **Stalker Portal** | Portal URL, STB MAC Address | `url` (Portal base), `stalker_mac`*, `stalker_token` (derived) | `u_res_0x7f13011b` (MAC Address) |

*For security and database standardization, XC credentials and Stalker MAC configurations are bound dynamically during API requests and encrypted or parsed in the native C++ player layer (`libtvplayer.so`).*

---

## 3. Network Validation & Verification Handshake

Upon clicking "Next", the application invokes an async validation request using the shared OkHttp network client. The app shows the message `Playlist updating…` (`u_res_0x7f130342`). 

```
                                  [ Setup Wizard Validation ]
                                               │
                       ┌───────────────────────┼───────────────────────┐
                       ▼                       ▼                       ▼
               [ M3U Ingestion ]        [ Xtream Codes ]      [ Stalker Portal ]
                Download header          Call API auth         Handshake auth
                       │                       │                       │
                       ▼                       ▼                       ▼
                 Verify first             Validate JSON         Parse session
                 char = #EXTM3U           profile response      handshake token
```

### Protocol Validation Routines

#### A. M3U / M3U8 Stream Inspection
The client does not load the file into memory. Instead, `mo4043(Uri uri, C3454 inputStream)` in [C1367.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p047/C1367.java) binds a `BufferedReader` to the input stream. It scans characters sequentially:
* **The EXTM3U Check**: It verifies that the very first character group matches the `#EXTM3U` header tag signature.
* **Stream Failures**: If the header is missing, parsing breaks instantly, saving network bandwidth and IO cycles on bad files.

#### B. Xtream Codes Profile Verification
The application makes a GET handshake request to the server:
```
GET /player_api.php?username=<user>&password=<pass>
```
* **Success Criteria**: The server must return an HTTP 200 with a valid JSON root containing a `user_info` child object (e.g. `auth: 1`, `status: "Active"`).
* **Connection Limits**: The client extracts configuration parameters like `max_connections` and writes them into local storage to alert the user if concurrent playback limits are breached.

#### C. Stalker Portal Handshake
The app calls the portal's MAC authentication endpoint:
```
POST /portal.php?type=stb&action=handshake
Cookie: mac=00:1A:79:XX:XX:XX; stb_lang=en;
```
* **Success Criteria**: The server validates the hardware MAC ID and returns a temporary authentication session token: `{"js":{"token":"<token_id>"}}`.

### Error Recovery State
If any validation fails (network timeout, wrong credentials, bad URL formatting), the wizard blocks transition and shows an error screen:
> *"An error has occurred while processing the playlist. Make sure you have entered the correct data and check your Internet connection."* (String: `u_res_0x7f13033d`).

---

## 4. EPG Source Binding

If verification succeeds, the wizard moves to the EPG configuration step.

* **M3U Playlists**: The wizard prompts the user:
  > *"Enter EPG URL for the playlist. You can add or change it later in the settings. XMLTV format is only supported."* (String: `u_res_0x7f13033b`).
* **XC / Stalker Portals**: EPG bindings are handled automatically. The client leverages the API structures to fetch listings (via `/player_api.php?action=get_short_epg` for XC or `/portal.php?action=get_short_epg` for Stalker) at runtime, skipping manual XMLTV configurations.

---

## 5. Database Commit & Background Ingestion

The user enters a **Playlist name** (`u_res_0x7f13033e`), defaulting to the hostname or "Playlist 1". Once the user confirms, the playlist is saved in the database.

```
                           [ Write Playlist Row to DB ]
                                       │
                         (PRAGMA journal_mode = WAL)
                                       │
                                       ▼
                       [ Spawn Background Worker Job ]
                          WorkManager: C3419.java
                                       │
                       ┌───────────────┴───────────────┐
                       ▼                               ▼
             [ Parse Categories ]             [ Parse Channel List ]
             Check group-title /            Batch commits of 500 rows
             get_live_categories            (withTransaction { ... })
```

### The Transaction Database Steps

1. **Insert Playlist Row**:
   Inserts configuration options into the `playlists` table in [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java). SQLite writes in **WAL (Write-Ahead Logging)** mode so active writes do not freeze EPG queries on the main thread.
2. **WorkManager Queue Enqueueing**:
   The app invokes [C3419.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p262/C3419.java) to schedule a background sync.
3. **Parse & Category Injection**:
   * For M3U, it iterates through lines, creating rows in `categories` for every unique `group-title=""`.
   * For XC, it pulls `/player_api.php?action=get_live_categories` and stores them.
4. **Batch Channel Commits**:
   To prevent slowing down the database during large imports, channels are accumulated in lists and written in batch transactions of **500 items** inside `withTransaction {}` blocks in [C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java).
5. **Completion Summary Screen**:
   The UI transitions to "Playlist is processed" (`u_res_0x7f13033f`) and shows parsed stats:
   > *"Channels: 8405, Movies: 1204, TV Shows: 450"*

---

## 6. Playback Viewport Loading & Zapping

When the user clicks "Done" (`u_res_0x7f13008e`), the wizard finishes and returns to the TV Guide layout.

### Loading the Viewport
* The UI queries the database for the active channels in the first group (`position_in_playlist ASC`).
* It loads the EPG programs into a 90-minute RAM cache (`LinkedHashMap` in EPG Controller) to enable 60fps horizontal scrolling on the TV Guide grid.
* The player automatically selects the first available channel in the list.

### Resolution & Buffering Pipeline

```
[ Tune Channel 101 ] ──► [ Check Playlist Type ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
        [ Direct Stream (M3U / XC) ]   [ Stalker Portal Stream ]
        Construct static URL:          Verify Token (playlists.stalker_token)
        server/live/u/p/101.ts                 │
                 │                             ▼
                 │                     Request temporary URL:
                 │                     /portal.php?action=create_link
                 │                             │
                 └──────────────┬──────────────┘
                                ▼
                   [ Handover to ExoPlayer ]
                   * Warm Connection Socket
                   * Buffer 500ms for play (Zap!)
                                │
                                ▼
                   [ Hardware Decoder Output ]
```

1. **URL Resolution**:
   * **M3U / XC**: Constructs the stream URL using stored credentials (e.g. `http://server/live/user/password/101.ts`).
   * **Stalker Portal**: Reads `stalker_token` from `playlists`. Resolves the command string (`ffmpeg http://localhost/ch/12345`) to a temporary link by calling:
     `/portal.php?type=itv&action=create_link&cmd=ffmpeg+http://localhost/ch/12345`
     Strips the `ffmpeg ` prefix and forwards the dynamic stream URL to ExoPlayer.
2. **ExoPlayer Buffering**:
   * ExoPlayer pulls connection streams using an OkHttp socket connection pool, avoiding fresh DNS and TLS setups.
   * `DefaultLoadControl` properties (in [C4654.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p392/C4654.java)) buffer data. Video starts rendering once `bufferForPlaybackMs` hits **500ms**.

---

## 7. Video Rendering & OSD Display

Finally, the player outputs decoded frames to the screen.

### The UI Stacking Order
The app uses a layered compositing approach to ensure the UI remains responsive even on low-end devices:
* **Layer 0 (Video Layer)**: Binds the player output to an Android `SurfaceView`. The GPU draws the video frames on a layer behind the main window, preventing UI guide refreshes from stuttering the stream.
* **Layer 1 (Legibility Scrim)**: Dynamically animates the alpha value from `0.0f` to `0.6f` to dim the video background for readability when menus are active.
* **Layer 2 (OSD Info Bar)**: Renders the channel overlay ([C3860.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p312/C3860.java)) containing the channel name, logo, current program EPG progress bar, and quality tags.

```
+--------------------------------------------------------+
|                                                        |
|                 [ Video Output Stream ]                |
|                                                        |
|  101 HBO HD  -  Westworld (S1 • E3)      09:20 - 10:20 |
|  [Logo]         Next: Game of Thrones    [====----]    |
+--------------------------------------------------------+
```

* **Zap Complete**: The Info Bar OSD fades out after 3 seconds of inactivity. The surf timer job completes, registering the channel tuning event in the database history table, and the user is now successfully **watching TV**.

---

## Architectural Comparison Matrix

| Step | M3U / M3U8 | Xtream Codes (XC) | Stalker Portal |
|---|---|---|---|
| **Form Fields** | URL input or Local file selector | Server URL, Username, Password | Portal URL, STB MAC Address |
| **Validation Action** | Sequential header parser (`#EXTM3U`) | REST API handshake profile verification | Portal STB Emulation handshake check |
| **EPG Retrieval** | User input EPG XMLTV URL | Automatic EPG JSON parsing | Automatic Stalker short-EPG sync |
| **Database Sync** | Streaming Line Tokenizer (Single pass) | Multi-action JSON categories/streams sync | Genre-scoped paginated API syncing |
| **Playback Resolution** | Direct stream URL mapping | Dynamic URL path compilation | Temporary signed `create_link` lookup |
| **Token Recovery** | N/A (Stateless URL) | N/A (Stateless Session) | Auto MAC token refresh on HTTP 403 |

---

## Summary of Speed Secrets Used in the Pipeline

* **Buffered XMLTV Scanning**: Only parses EPG entries containing channel IDs that are actively mapped to active playlists, skipping the rest of the XML file.
* **Epoch Seconds indexing**: Uses narrow integer values (Epoch seconds) in SQLite indices rather than long milliseconds, accelerating the database query checks during vertical/horizontal guides.
* **OkHttp Socket Pooling**: Reuses network sockets during channel swaps (zapping), avoiding connection handshake delays.
* **Low Buffering thresholds**: Tunes ExoPlayer to trigger playback when only 500ms of data is buffered, enabling near-instant zapping.
