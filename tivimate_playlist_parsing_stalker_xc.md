# TiViMate Playlist Deep-Dive — M3U, Stalker Portal & Xtream Codes Architecture

This document provides a highly detailed architectural analysis of **TiViMate’s playlist ingestion and mapping engines**. It breaks down the exact parsing rules, API handshakes, and database structures used to ingest **M3U/M3U8** files, communicate with **Xtream Codes (XC)** endpoints, and emulate MAG boxes on **Stalker Portals** securely and efficiently.

---

## 1. M3U / M3U8 INGESTION & ATTRIBUTE EXTRACTION

An M3U file is simple plain text. However, IPTV providers embed complex television metadata directly inside the `#EXTINF` line. TiViMate parses these using a streaming line reader, extracting attributes dynamically and mapping them into relational tables.

```
[ Raw M3U String Ingestion ]
  │
  ▼ (Reads line-by-line via BufferedReader)
#EXTINF:-1 tvg-id="CNN.us" tvg-logo="http://logo.com/cnn.png" group-title="News",CNN US
http://server.com/live/cnn.ts
  │
  ▼ (Attribute Tokenization Scanner)
* tvg-id      ──► Bind to XMLTV EPG target mapping
* tvg-logo    ──► Cache via Glide logo disk factory (250MB)
* group-title ──► Relate to category row (Insert if not exists)
* stream name ──► Set as rawName (CNN US)
* stream URL  ──► Write as streaming url path in channel table
```

### The Key `#EXTINF` Parsed Attributes

*   **`tvg-id`**: The primary key used to link this channel with EPG database records inside `tvg_programs`.
*   **`tvg-logo`**: The image asset URL. Extracted and passed straight to the LRU Glide cache manager.
*   **`group-title`**: Defines the category name. If a category with this name does not exist under the current `playlist_id`, a new row is created in `categories` and its primary key is bound to the channel's `category_id`.
*   **`catchup="default"` or `catchup="shift"`**: Flags this channel as supporting timeshift.
*   **`catchup-days="7"`**: Configures the EPG timeline boundaries to allow scrolling left in time for 7 full days.
*   **`catchup-source="..."`**: Stores the custom URL template used to request historical timeshift packets.

---

## 2. XTREAM CODES (XC) API CLIENT ENGINE

Xtream Codes is a database-backed IPTV billing and streaming panel. Instead of downloading a massive text file, the client issues structured HTTP API queries. TiViMate translates these REST responses directly into SQLite entries.

```
                  [ TV App Startup / Sync WorkManager ]
                                    │
                         (HTTPS HTTP Request Handshake)
  /player_api.php?username=user&password=pass ──────────────────────────┐
                                                                        ▼
  /player_api.php?username=...&password=...&action=get_live_categories  ├──► Parse JSON Responses
  /player_api.php?username=...&password=...&action=get_live_streams     ├──► Write to SQLite Database
  /player_api.php?username=...&password=...&action=get_vod_categories   ├──► Map categories & channels
  /player_api.php?username=...&password=...&action=get_vod_streams      │
                                                                        ▼
                                                              [ Complete Local Cache ]
```

### The 4-Step XC Sync Sequence

1.  **Authentication Handshake**:
    The client calls `/player_api.php?username=...&password=...`.
    *   The server returns a large JSON object containing `user_info` (expiration date, active connection count, max allowed connections) and `server_info` (server timezone, active HTTPS ports).
2.  **Category Mapping**:
    The client calls `/player_api.php?username=...&action=get_live_categories` and `action=get_vod_categories`.
    *   Returns a JSON array: `[{"category_id":"1","category_name":"News","parent_id":"0"}]`.
    *   These are written directly to the SQLite `categories` table.
3.  **Stream Mapping**:
    The client calls `/player_api.php?username=...&action=get_live_streams` or `action=get_vod_streams`.
    *   Returns detailed channel lists with stream IDs: `{"num":1,"name":"CNN","stream_id":41020,"stream_icon":"...","category_id":"1"}`.
    *   The client maps these on-the-fly to streaming URLs: `${server}/live/${username}/${password}/${stream_id}.ts` for live, or `${server}/movie/${username}/${password}/${stream_id}.mp4` for VOD, saving them in the `channels` and `movies` tables.
4.  **Short EPG Sync**:
    To populate the Channel Bar instantly without running full XMLTV updates, the client queries:
    `/player_api.php?username=...&action=get_short_epg&stream_id=...`
    This parses the active and subsequent program details in milliseconds, keeping zapping metadata current.

---

## 3. STALKER PORTAL (MAG ENGINE EMULATION)

Stalker portals are highly restrictive, closed APIs designed for hardware MAG boxes. Integrating them requires emulating a hardware box's handshake lifecycle and managing cookie headers.

### The Stalker Handshake & URL Lifecycle

MAG boxes communicate via `/portal.php`. The client must emulate this sequence using unique custom headers:

```
                  [ Stalker Emulation Client ]
                                │
                  (Handshake: Sends MAC Address)
                  /portal.php?type=itv&action=handshake&mac=00:1A:79...
                                │
                                ▼
                  Returns: {"js":{"token":"9cfb3b..."}} 
                  (Persists token transactionally in SQLite)
                                │
                                ▼
                  (Fetch Genres & Channel Indices)
                  /portal.php?type=itv&action=get_ordered_iptv_genres
                  /portal.php?type=itv&action=get_all_channels
                                │
                                ▼
                  (Play: Generate Temp Streaming Link)
                  /portal.php?type=itv&action=create_link&cmd=ffmpeg+http://...
                                │
                                ▼
                  Returns: {"js":{"cmd":"http://server.com/live/123/stream.ts"}}
                  (ExoPlayer binds direct temporary link)
```

1.  **MAC Authentication**:
    The client initiates a handshake by sending a unique MAG MAC Address (typically starting with `00:1A:79:`):
    `/portal.php?type=itv&action=handshake&mac=00:1A:79:XX:XX:XX`
    *   The server returns a dynamic session `token`. This token is stored in the database `playlists.stalker_token` column (`C3220.java` line 172).
2.  **Category & Channel Syncing**:
    *   **Genres**: `/portal.php?type=itv&action=get_ordered_iptv_genres` resolves category lists.
    *   **Channels**: `/portal.php?type=itv&action=get_all_channels` fetches the complete stream list, mapping the stalker ID variables onto standard relational tables.
3.  **Temporary Playback Link Resolving**:
    Stalker portals do not permit direct streaming. To play a channel, the client must request a temporary, time-locked link:
    `/portal.php?type=itv&action=create_link&cmd=ffmpeg+http://localhost/ch_id`
    *   The server returns a dynamic, signed stream URL.
    *   **Silent Token Auto-Refresh**: If ExoPlayer encounters an active network drop or HTTP `403 Forbidden` error (indicating the stalker session token has expired), the client catches the error, triggers the MAC handshake in the background, updates `stalker_token` in the database, re-fetches the temporary playback link, and resumes ExoPlayer playback seamlessly without halting the media engine.
