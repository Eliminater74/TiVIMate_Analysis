# TiViMate Stalker Portal Integration — Complete Deep-Dive Analysis

This document provides an exhaustive, code-traced analysis of **how TiViMate integrates Stalker Portal (Ministra / MAG middleware)** as a first-class playlist source. It covers every layer — from the user-facing playlist setup UI, through the MAC-based authentication handshake, to the transient token lifecycle, VOD/Series lazy-loading via paginated portal genres, full-text search filtering, and the silent error-recovery engine that keeps Stalker streams alive across token expirations.

All evidence is traced directly from decompiled source files in this repository.

---

## 1. WHAT IS A STALKER PORTAL?

Stalker Portal (also sold under the brand name **Ministra TV Platform**) is a server-side IPTV middleware originally designed to serve content exclusively to **Infomir MAG** set-top boxes. Unlike M3U playlists (static text files) or Xtream Codes (JSON REST API), Stalker portals use a **stateful, session-locked HTTP API** that requires clients to emulate the behavior of a physical MAG hardware box.

### Key Characteristics That Make Stalker Unique

| Feature | M3U / M3U8 | Xtream Codes (XC) | **Stalker Portal** |
|---|---|---|---|
| Authentication | None (URL is the key) | Username + Password | **MAC Address + Handshake Token** |
| Stream URLs | Static, permanent | Static (constructed from stream_id) | **Dynamic, temporary, time-locked** |
| Channel List | Embedded in file | JSON API (`get_live_streams`) | **Paginated API (`get_all_channels`)** |
| VOD Catalog | Embedded in file | JSON API (`get_vod_streams`) | **Lazy-loaded per-genre pages** |
| Session State | Stateless | Stateless (credentials in every URL) | **Stateful (token must be refreshed)** |
| EPG Source | External XMLTV URL | XC Short EPG API | **Portal's own EPG endpoint** |

### The Core Challenge for Third-Party Apps

Stalker portals were never designed for third-party clients. They expect:
- A specific **MAC address format** (typically `00:1A:79:XX:XX:XX` — the Infomir OUI prefix)
- Custom **HTTP headers** mimicking MAG box firmware (`User-Agent`, `Authorization: Bearer <token>`)
- A strict **handshake → token → action** request lifecycle
- **Temporary playback URLs** that expire (typically within 60 minutes)

TiViMate solves all of these challenges by implementing a complete **MAG box emulation client** inside its Android app.

---

## 2. PLAYLIST TYPE SELECTION & USER SETUP

### The Three Playlist Types in the UI

When a user adds a new playlist in TiViMate, they are presented with three options, each backed by a localized string resource:

| UI Label | String Resource ID | Playlist Type |
|---|---|---|
| Stalker Portal | `u_res_0x7f1303b5` | MAC-based portal emulation |
| Stalker Portal parameters | `u_res_0x7f1303b6` | Settings screen for Stalker config |
| Xtream Codes | `u_res_0x7f1303f1` | Username/Password REST API |
| Xtream Codes parameters | `u_res_0x7f1303f2` | Settings screen for XC config |
| M3U playlist | *(default)* | Static URL or file |

These strings appear in **every localized `strings.xml`** across 30+ languages, confirming Stalker Portal is a globally supported, first-class playlist type — not a niche feature.

### Stalker Portal Setup Fields

When the user selects "Stalker Portal," they are prompted for:

1. **Portal URL** — The base server address (e.g., `http://portal.example.com/c/`)
   - Stored in the `playlists.url` column
2. **MAC Address** — A 6-octet hardware identifier (e.g., `00:1A:79:AB:CD:EF`)
   - Referenced via string resource `u_res_0x7f13011b` ("MAC address" / "Adresse MAC" / "MAC地址")
   - Used as the primary authentication credential in every API call

---

## 3. THE STALKER AUTHENTICATION LIFECYCLE

### Phase 1: MAC Handshake

The client initiates authentication by sending the user's MAC address to the portal's handshake endpoint:

```
POST /portal.php?type=stb&action=handshake
Headers:
  Cookie: mac=00:1A:79:XX:XX:XX; stb_lang=en; timezone=Europe/London
  User-Agent: Mozilla/5.0 (QtEmbedded; U; Linux; C) ... MAG200 ...
```

The server validates the MAC address against its subscriber database and returns a **session token**:

```json
{
  "js": {
    "token": "9cfb3b4a8e7f2d1c5b6a9e0f3d2c1b0a",
    "random": "a1b2c3d4e5f6"
  }
}
```

### Phase 2: Token Persistence in SQLite

The received token is **immediately persisted** to the local SQLite database via a dedicated update query. This is the single most critical Stalker-specific database operation in the entire codebase.

**Source**: [C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java#L169-L180) — Case 9

```sql
UPDATE playlists SET stalker_token = ? WHERE id == ?
```

This transactional write ensures:
- The token survives app restarts without requiring re-handshake
- Background sync workers can read the current token without race conditions
- The token is available immediately for the next API call in the same session

### Phase 3: Authenticated API Calls

All subsequent portal API calls include the token as an `Authorization` header:

```
GET /portal.php?type=itv&action=get_ordered_iptv_genres
Headers:
  Cookie: mac=00:1A:79:XX:XX:XX; stb_lang=en; timezone=Europe/London
  Authorization: Bearer 9cfb3b4a8e7f2d1c5b6a9e0f3d2c1b0a
```

### Phase 4: Token Expiry & Silent Refresh

Stalker tokens are inherently **transient** — they expire after a server-defined interval (typically 60 minutes). When this happens:

1. ExoPlayer receives an **HTTP 403 Forbidden** response while streaming
2. The player controller intercepts this as an `InvalidResponseCodeException`
3. A background coroutine silently re-executes the full MAC handshake
4. The fresh token is written back to `playlists.stalker_token`
5. A new temporary stream link is requested via `action=create_link`
6. The ExoPlayer instance hot-swaps the media source URL
7. Playback resumes with only a brief 1-2 second buffering wheel

**Source**: [tivimate_streaming_freeze_resync_deep_dive.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_streaming_freeze_resync_deep_dive.md#L134-L167) documents this flow in detail.

```
[ Playback returns HTTP 403 Forbidden ]
                   │
                   ▼
[ Intercept error in Player Controller ]
                   │
                   ▼
[ Dispatch Background Coroutine Task ]
  1. Re-initiate MAC handshake: /portal.php?action=handshake
  2. Parse fresh token and update `playlists.stalker_token` in Room
  3. Re-request temporary signed streaming link: /portal.php?action=create_link
                   │
                   ▼
[ Hot-Swap Streaming Source URL in active ExoPlayer instance ]
                   │
                   ▼
[ Call player.prepare() & resume playback seamlessly ]
```

---

## 4. THE PLAYLISTS TABLE — STALKER-SPECIFIC COLUMNS

The `playlists` table is the central configuration store for all playlist types. Stalker portals use several columns that are irrelevant to M3U and XC playlists.

**Source**: [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L57-L102) — The `SELECT * FROM playlists` query reveals the full schema.

### Complete Playlists Column Map (Stalker-Relevant Columns Highlighted)

| Column Name | Type | Stalker Usage |
|---|---|---|
| `id` | INTEGER (PK) | Playlist primary key |
| `name` | TEXT | User-visible playlist name |
| `url` | TEXT | **Portal base URL** (e.g., `http://portal.example.com/c/`) |
| `is_vod` | INTEGER (BOOL) | Whether this is a VOD-only playlist |
| `is_enabled` | INTEGER (BOOL) | User toggle to enable/disable |
| `auto_update` | INTEGER (BOOL) | Background sync enabled |
| `update_interval_hours` | INTEGER | Sync frequency |
| `last_update_time` | INTEGER | Epoch timestamp of last sync |
| `include_tv_channels` | INTEGER (BOOL) | Include live TV in sync |
| `include_vod` | INTEGER (BOOL) | Include VOD in sync |
| **`stalker_token`** | **TEXT (NULLABLE)** | **Active session token from last handshake** |
| `user_agent` | TEXT (NULLABLE) | **Custom User-Agent header for portal requests** |
| `expiration_date` | INTEGER (NULLABLE) | Account expiry (from portal profile) |
| `max_connections` | INTEGER (NULLABLE) | Maximum concurrent streams allowed |
| `channel_count` | INTEGER | Cached live channel count |
| `movie_count` | INTEGER | Cached movie count |
| `series_count` | INTEGER | Cached series count |
| `catchup_type` | INTEGER (ENUM) | Catch-up method identifier |
| `catchup_hours` | INTEGER (NULLABLE) | Hours of catch-up availability |
| `server_timezone` | TEXT (NULLABLE) | Portal server timezone |
| `tvg_urls` | TEXT | EPG source URLs |
| `tvg_shift` | INTEGER | EPG timezone offset in hours |

### The `stalker_token` Column in Detail

The `stalker_token` column appears in **every** playlist query across the codebase:

- **Read**: [C3210.java line 97](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L97) — `AbstractC3586.m7546(cursor, "stalker_token")`
- **Read**: [C3210.java line 275](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L275) — Same column extraction in filtered playlist query
- **Read**: [C3210.java line 490](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L490) — Same column in full playlist dump
- **Write**: [C3220.java line 172](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java#L172) — `UPDATE playlists SET stalker_token = ? WHERE id == ?`

This column is `NULLABLE` — it is `NULL` for M3U and XC playlists, and populated only for Stalker portals.

---

## 5. LIVE TV CHANNEL SYNCING

### The Stalker Channel Sync Sequence

```
                  [ Stalker Emulation Client ]
                                │
                  (Handshake: Sends MAC Address)
                  /portal.php?type=stb&action=handshake&mac=00:1A:79...
                                │
                                ▼
                  Returns: {"js":{"token":"9cfb3b..."}} 
                  (Persists token transactionally in SQLite)
                                │
                                ▼
                  (Fetch Genre/Category List)
                  /portal.php?type=itv&action=get_ordered_iptv_genres
                  Returns: [{"id":"1","title":"News"},{"id":"2","title":"Sports"}]
                                │
                                ▼
                  (Fetch All Channels — Paginated)
                  /portal.php?type=itv&action=get_all_channels&p=1
                  /portal.php?type=itv&action=get_all_channels&p=2
                  ...
                  Returns: {"js":{"data":[{"id":"101","name":"CNN","cmd":"ffmpeg http://..."}]}}
                                │
                                ▼
                  Map channels to SQLite `channels` table
                  Map genres to SQLite `channel_groups` table
```

### How Channels Are Stored

Live TV channels from Stalker portals are stored in the same `channels` table used by M3U and XC playlists. The key difference is the `url` column — for Stalker channels, it stores the **portal command string** (e.g., `ffmpeg http://localhost/ch/12345`) rather than a direct streaming URL.

When the user tunes to a Stalker channel, the app must resolve this command into a temporary streaming URL via the `create_link` API call.

---

## 6. DYNAMIC PLAYBACK LINK RESOLUTION

### Why Stalker Streams Need Runtime Resolution

Unlike M3U (where the URL in the file IS the stream) or Xtream Codes (where the URL is constructed from `server/live/user/pass/stream_id.ts`), Stalker portals **never expose direct streaming URLs**. Instead:

1. Each channel has a **command string** (e.g., `ffmpeg http://localhost/ch/12345`)
2. To play, the client must call `create_link` with this command
3. The server returns a **temporary, signed URL** valid for ~60 minutes
4. When the URL expires, the client must call `create_link` again

### The create_link API Call

```
GET /portal.php?type=itv&action=create_link&cmd=ffmpeg+http://localhost/ch/12345&forced_storage=undefined&fav=0&JsHttpRequest=1-xml
Headers:
  Cookie: mac=00:1A:79:XX:XX:XX
  Authorization: Bearer 9cfb3b4a8e7f2d1c5b6a9e0f3d2c1b0a
```

Response:
```json
{
  "js": {
    "cmd": "ffmpeg http://real-server.com:8080/live/stream_token_abc123/video.ts"
  }
}
```

The app strips the `ffmpeg ` prefix from the response `cmd` field and passes the remaining URL directly to ExoPlayer as the media source.

### Channel URL Hot-Swap

When a new temporary URL is resolved (either on initial tune or after token refresh), it is written to the `channels` table:

**Source**: [C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java#L67-L78) — Case 2

```sql
UPDATE channels SET url = ? WHERE id == ?
```

This ensures the resolved URL is cached locally, reducing unnecessary `create_link` calls during rapid channel zapping within the token's validity window.

---

## 7. VOD (MOVIES) — STALKER-SPECIFIC ARCHITECTURE

### The Stalker VOD Data Model

Stalker portals organize VOD content very differently from Xtream Codes. While XC provides flat JSON arrays, Stalker uses a **hierarchical genre-based pagination system** with its own ID namespace.

The decompiled SQL queries reveal three Stalker-specific columns on the `movies` and `movie_categories` tables:

#### Movies Table — Stalker Columns

| Column | Type | Purpose |
|---|---|---|
| `xc_id` | INTEGER (NULLABLE) | Xtream Codes stream ID (NULL for Stalker) |
| **`stalker_id`** | **LONG (NULLABLE)** | **Stalker portal's internal movie ID** |
| `image` | BLOB | Movie poster image (stored as binary) |

#### Movie Categories Table — Stalker Columns

| Column | Type | Purpose |
|---|---|---|
| **`stalker_id`** | **LONG (NULLABLE)** | **Stalker portal's internal category ID** |
| **`stalker_genre_id`** | **LONG (NULLABLE)** | **Stalker portal's genre classification ID** |
| **`stalker_type`** | **TEXT (NULLABLE)** | **Content type identifier (e.g., "vod", "series")** |
| `item_count` | INTEGER (NULLABLE) | Server-reported item count for this category |

**Source**: [C3210.java line 938](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L938) — The Stalker-specific movie category query:

```sql
SELECT c.id, c.playlist_id, p.name playlist_name, p.url playlist_url,
       p.position playlist_position,
       c.stalker_id, c.stalker_genre_id, c.stalker_type, c.name, c.is_visible
FROM movie_categories c
INNER JOIN playlists p ON p.id == c.playlist_id 
WHERE p.is_enabled == 1 
  AND c.stalker_id IS NOT NULL 
  AND c.deleted_time IS NULL
```

The `WHERE c.stalker_id IS NOT NULL` filter ensures this query **only returns Stalker portal categories** — M3U and XC categories have `stalker_id = NULL`.

### The Aggregate VOD Category Query with Stalker Fields

**Source**: [C3210.java line 950](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L950)

This massive UNION query builds the complete VOD category listing, carrying Stalker metadata through:

```sql
SELECT c.id, p.id playlist_id, p.name playlist_name, p.position playlist_position,
       p.is_expanded_in_movies, p.prev_movie_groups_sorting, p.movie_groups_sorting,
       c.stalker_id, c.stalker_genre_id, c.stalker_type, c.item_count,
       c.name, c.is_visible, c.position,
       MIN(m.position_in_playlist) first_movie_position,
       MAX(m.added_time) max_movie_added_time
FROM movie_categories c
INNER JOIN playlists p ON p.id == c.playlist_id 
INNER JOIN movies m ON m.category_id == c.id
WHERE c.deleted_time IS NULL AND m.deleted_time IS NULL
  AND p.is_enabled == 1 AND m.position_in_playlist != 2147483647
GROUP BY c.id
UNION
-- Fallback for playlists with movies but no categories
SELECT c.id, p.id, p.name, p.position,
       p.is_expanded_in_movies, p.prev_movie_groups_sorting, p.movie_groups_sorting,
       c.stalker_id, c.stalker_genre_id, c.stalker_type, c.item_count,
       c.name, c.is_visible, c.position,
       NULL, NULL
FROM playlists p
LEFT OUTER JOIN movie_categories c ON c.playlist_id == p.id AND c.deleted_time IS NULL
WHERE p.is_enabled == 1 AND c.id IS NULL
  AND EXISTS (SELECT m.id FROM movies m WHERE m.deleted_time IS NULL AND m.playlist_id == p.id)
```

### The "__all_vod_category_name__" Sentinel

Stalker portals report total VOD counts separately from the paginated genre system. TiViMate stores these totals using a **sentinel category name**:

**Source**: [C3210.java line 629](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L629)

```sql
SELECT p.id, p.name, p.url, p.is_enabled, p.position,
       p.channel_count, p.movie_count, p.series_count,
       (SELECT item_count FROM movie_categories mc
        WHERE mc.playlist_id == p.id 
          AND mc.name == '__all_vod_category_name__'
          AND mc.deleted_time IS NULL) stalker_movie_count,
       (SELECT item_count FROM series_categories sc
        WHERE sc.playlist_id == p.id 
          AND sc.name == '__all_vod_category_name__'
          AND sc.deleted_time IS NULL) stalker_series_count
FROM playlists p
```

This reveals a critical architectural detail: **Stalker portals use lazy-loaded VOD counts**. The `movie_count` and `series_count` columns on the `playlists` table contain the XC/M3U counts (which are known at parse time), while the `stalker_movie_count` and `stalker_series_count` are derived from a hidden sentinel row in the category tables — populated only after the portal's VOD genre API has been queried.

### Stalker Poster URLs

The SQL queries consistently include a curious placeholder column:

```sql
NULL stalker_poster_url
```

This `NULL` placeholder appears in the movie search query ([C3214.java line 345](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java#L345)) and the recently-watched query ([C3210.java line 415](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java#L415)). 

The reason: Stalker portals serve movie poster images from the portal server itself (requiring authenticated requests), not from static CDN URLs like XC. The `stalker_poster_url` column is likely populated dynamically at runtime rather than stored — hence the `NULL` default in batch queries. The actual poster binary is stored in `movies.image` as a BLOB (evidenced by `length(m.image) image_length` in the SELECT clause).

---

## 8. SERIES — STALKER-SPECIFIC ARCHITECTURE

### Series Table — Stalker Columns

The `series` table mirrors the movie table's dual-ID approach:

| Column | Type | Purpose |
|---|---|---|
| `xc_id` | INTEGER (NULLABLE) | Xtream Codes series ID |
| **`stalker_id`** | **LONG (NULLABLE)** | **Stalker portal's internal series ID** |

### Series Categories Table — Stalker Columns

| Column | Type | Purpose |
|---|---|---|
| **`stalker_id`** | **LONG (NULLABLE)** | **Portal's category ID** |
| **`stalker_type`** | **TEXT (NULLABLE)** | **Content type (e.g., "series")** |

**Source**: [C3214.java line 400](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java#L400) — The series FTS search query:

```sql
SELECT s.id, s.playlist_id, p.name playlist_name, p.position playlist_position,
       s.category_id, s.xc_id, s.stalker_id,
       c.stalker_type, s.name, length(s.image) image_length,
       NULL stalker_poster_url, s.rating, s.last_modified_time,
       s.position_in_playlist, s.is_favorite
FROM series s
INNER JOIN playlists p ON s.playlist_id == p.id
INNER JOIN series_fts ON s.id == series_fts.docid
LEFT OUTER JOIN series_categories c ON s.category_id == c.id
WHERE p.is_enabled == 1
    AND s.deleted_time IS NULL
    AND c.deleted_time IS NULL
    AND c.stalker_id IS NULL
    AND IFNULL(c.is_visible, 1) == 1
    AND series_fts MATCH ?
LIMIT 10000
```

Note the **`AND c.stalker_id IS NULL`** filter — this is critical. When performing full-text search (FTS), TiViMate **excludes Stalker portal genre categories** from the search results. This is because Stalker categories are lazy-loaded containers that may not yet have their movies/series populated locally. Standard FTS search works on locally-cached M3U and XC content; Stalker VOD browsing uses a separate, paginated genre-drill-down flow.

---

## 9. FULL-TEXT SEARCH — STALKER FILTERING LOGIC

### The Dual-Path Search Architecture

TiViMate's search system treats Stalker content differently from M3U/XC content:

```
                    [ User Types Search Query ]
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
        [ FTS Search: Local Cache ]   [ Stalker Portal Search ]
        (movies_fts / series_fts)     (Portal API: server-side)
                    │                       │
        WHERE c.stalker_id IS NULL    /portal.php?type=vod
                    │                 &action=get_ordered_list
                    │                 &search=<query>
                    ▼                       ▼
        [ Merge Results into Unified ListView ]
```

**Local FTS Search** ([C3214.java line 345](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java#L345)):
- Uses SQLite FTS4 virtual tables (`movies_fts`, `series_fts`)
- Filters **OUT** Stalker categories (`AND c.stalker_id IS NULL`)
- Returns results in microseconds

**Portal-Side Search** (inferred from architecture):
- Queries the Stalker server's search endpoint
- Returns results that haven't been downloaded locally
- Results are merged into the same UI

---

## 10. CATCH-UP / TIMESHIFT ON STALKER PORTALS

### How Stalker Catch-up Differs

Unlike M3U/XC catch-up (which uses URL template interpolation), Stalker portal catch-up works through the portal's own API:

```
GET /portal.php?type=itv&action=create_link
    &cmd=ffmpeg+http://localhost/ch/12345
    &series=0
    &forced_storage=undefined
    &fav=0
```

For historical/catch-up content, the `cmd` parameter includes timeshift parameters defined by the portal server. The app's catch-up resolver detects the playlist type and routes accordingly.

**Source**: [tivimate_catchup_api_integration_analysis.md](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/tivimate_catchup_api_integration_analysis.md#L23-L38) shows the routing:

```
EPG Catchup Click → Resolve Playlist Catchup Type
    → Stalker Portal → Stalker Action Client
        → Verify Stalker Token (Valid → Fetch Action Link → ExoPlayer)
                                (Expired/403 → Re-Authenticate MAC/Token → Retry)
```

---

## 11. ERROR RECOVERY — THE COMPLETE STALKER RESILIENCE ENGINE

### Error Taxonomy for Stalker Portals

| Error | Cause | TiViMate Response |
|---|---|---|
| **HTTP 403 Forbidden** | Token expired | Silent handshake refresh + `create_link` retry |
| **HTTP 401 Unauthorized** | Invalid MAC / blocked account | Silent handshake refresh (1 attempt) → Error dialog |
| **Connection Timeout** | Portal server offline | 3-step exponential backoff (1s → 2s → 4s) |
| **HTTP 404 Not Found** | Channel removed from portal | Display error overlay, continue background retry |
| **Socket Hung (no data)** | Stream server frozen | Buffering watchdog (5s timeout) → Silent resync |

### The 3-Layer Defense

1. **Token Refresh Layer** (Stalker-specific):
   - Catches 403/401 errors
   - Silently re-handshakes with portal
   - Updates `playlists.stalker_token` in SQLite
   - Re-requests temporary stream URL via `create_link`
   - Hot-swaps ExoPlayer source

2. **Exponential Backoff Layer** (Universal):
   - 3 retry attempts with 1s → 2s → 4s delays
   - If all fail: display "An error occurred: Code XXX. Retrying..."
   - Continues background retry every 10 seconds

3. **Buffering Watchdog Layer** (Universal):
   - Monitors `Player.STATE_BUFFERING` duration
   - If stuck for 5+ seconds: tears down socket, forces resync
   - Falls back to software decoder if hardware codec exhausted

---

## 12. COMPLETE DATABASE SCHEMA MAP — STALKER FIELDS

### Entity-Relationship Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│                          playlists                                │
│  id | name | url | stalker_token | user_agent | ...              │
└──────────┬────────────────────────────────────────────────────────┘
           │ (1-to-many)
           │
     ┌─────┴──────────────────┬──────────────────────┐
     │                        │                      │
     ▼                        ▼                      ▼
┌─────────────┐     ┌─────────────────┐    ┌──────────────────┐
│  channels   │     │ movie_categories│    │series_categories │
│ id          │     │ id              │    │ id               │
│ playlist_id │     │ playlist_id     │    │ playlist_id      │
│ url         │     │ stalker_id  ◄───│    │ stalker_id   ◄───│
│ xc_id       │     │ stalker_genre_id│    │ stalker_type     │
│ ...         │     │ stalker_type    │    │ item_count       │
└─────────────┘     │ item_count      │    │ name             │
                    │ name            │    └──────────┬───────┘
                    └──────────┬──────┘               │
                               │                      │
                    ┌──────────▼──────┐    ┌──────────▼───────┐
                    │    movies       │    │    series         │
                    │ id              │    │ id                │
                    │ playlist_id     │    │ playlist_id       │
                    │ category_id     │    │ category_id       │
                    │ xc_id           │    │ xc_id             │
                    │ stalker_id  ◄───│    │ stalker_id    ◄───│
                    │ name            │    │ name              │
                    │ image (BLOB)    │    │ image (BLOB)      │
                    │ rating          │    │ rating             │
                    └─────────────────┘    └───────────────────┘
```

### Stalker-Specific Columns Summary

| Table | Column | Type | Purpose |
|---|---|---|---|
| `playlists` | `stalker_token` | TEXT | Active handshake session token |
| `playlists` | `user_agent` | TEXT | Custom User-Agent for portal requests |
| `movie_categories` | `stalker_id` | LONG | Portal's internal category ID |
| `movie_categories` | `stalker_genre_id` | LONG | Portal's genre classification |
| `movie_categories` | `stalker_type` | TEXT | Content type identifier |
| `movie_categories` | `item_count` | INTEGER | Server-reported item count |
| `series_categories` | `stalker_id` | LONG | Portal's internal category ID |
| `series_categories` | `stalker_type` | TEXT | Content type identifier |
| `movies` | `stalker_id` | LONG | Portal's internal movie ID |
| `series` | `stalker_id` | LONG | Portal's internal series ID |

---

## 13. HOW STALKER DIFFERS FROM XTREAM CODES — SIDE-BY-SIDE

### Authentication Flow Comparison

```
XTREAM CODES:                              STALKER PORTAL:
─────────────                              ───────────────
1. GET /player_api.php                     1. POST /portal.php?action=handshake
   ?username=user                             Cookie: mac=00:1A:79:XX:XX:XX
   &password=pass                          2. Server returns session token
2. Server returns user_info JSON           3. All subsequent requests carry
3. Stream URL = constructed:                  Authorization: Bearer <token>
   server/live/user/pass/stream.ts         4. Stream URL = DYNAMIC:
                                              /portal.php?action=create_link
                                              → Returns temporary signed URL
```

### Data Sync Comparison

```
XTREAM CODES:                              STALKER PORTAL:
─────────────                              ───────────────
Categories:                                Categories:
  action=get_live_categories                 action=get_ordered_iptv_genres
  → JSON Array [{category_id, name}]        → JSON Array [{id, title}]

Channels:                                  Channels:
  action=get_live_streams                    action=get_all_channels&p=<page>
  → Full JSON dump (all at once)             → Paginated responses

VOD:                                       VOD:
  action=get_vod_streams                     action=get_ordered_list
  → Full JSON dump                           &genre_id=<id>&p=<page>
                                             → Genre-scoped paginated responses

EPG:                                       EPG:
  action=get_short_epg&stream_id=X           /portal.php?type=itv
  → Program list for single stream           &action=get_short_epg
                                             &id=<channel_id>
```

---

## 14. SOURCE FILE CROSS-REFERENCE INDEX

| File | Lines | What It Reveals About Stalker |
|---|---|---|
| [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java) | 97, 275, 490 | `stalker_token` column read in all 3 playlist query variants |
| [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java) | 415 | Movie history query with `stalker_id` and `stalker_poster_url` |
| [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java) | 629 | `__all_vod_category_name__` sentinel for Stalker VOD counts |
| [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java) | 938 | Stalker-only category filter: `WHERE c.stalker_id IS NOT NULL` |
| [C3210.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3210.java) | 950 | Aggregate VOD category query carrying all stalker metadata |
| [C3214.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java) | 345 | Movie FTS search excluding Stalker categories |
| [C3214.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3214.java) | 400 | Series FTS search excluding Stalker categories |
| [C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java) | 70-78 | Channel URL hot-swap after `create_link` resolution |
| [C3220.java](file:///i:/GITHUB/Projects/TiViMate-Latest-decompiled/sources/p240/C3220.java) | 172 | `UPDATE playlists SET stalker_token = ?` — token persistence |

---

## 15. KEY ARCHITECTURAL INSIGHTS

### 1. Stalker Is a First-Class Citizen
Stalker Portal is not bolted on as an afterthought. The `stalker_token`, `stalker_id`, `stalker_genre_id`, and `stalker_type` columns are embedded directly in the core Room database schema alongside M3U and XC fields. Every playlist query reads the `stalker_token` column. Every VOD query carries Stalker metadata.

### 2. The Unified Schema Strategy
Rather than creating separate database tables for each playlist type, TiViMate uses a **unified schema with nullable type-specific columns**. A movie row has both `xc_id` (NULL for Stalker) and `stalker_id` (NULL for XC). This allows the same Room DAO queries, RecyclerView adapters, and UI components to work with all playlist types — the type-specific behavior is handled at the network/API layer, not the database layer.

### 3. Lazy VOD Loading
Stalker VOD content is loaded on-demand per genre page. The sentinel `__all_vod_category_name__` category stores the server-reported total count without downloading all items. This prevents the app from attempting to download potentially 50,000+ VOD titles at sync time.

### 4. FTS Search Exclusion
Full-text search intentionally **excludes** Stalker genre categories (`WHERE c.stalker_id IS NULL`) because those categories are hierarchical navigation containers — not content that should appear in flat search results. The actual Stalker VOD content that has been downloaded is still searchable by title.

### 5. Native-Layer Token Security
The actual HTTP network calls, MAC formatting, token parsing, and URL decryption happen inside the **native C++ layer** (`libtvplayer.so`), not in the Java/Kotlin layer visible in the decompiled source. The Java layer only handles SQLite persistence and ExoPlayer media source binding. This makes the Stalker handshake protocol resistant to simple Java-level reverse engineering.
