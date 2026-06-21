# TiViMate Logo Resolution Pipeline & Caching Architecture

This document analyzes how TiViMate resolves, prioritizes, caches, and renders channel and VOD logos (icons/posters). It is based on database schema extraction (`C3210.java`) and string resource analysis.

---

## 1. THE LOGO HIERARCHY (PRIORITY SYSTEM)

TiViMate aggregates logo data from multiple sources: the M3U playlist file, XMLTV EPG data, and local storage folders. Because these sources frequently conflict or provide broken links, the app implements a **Logo Priority System**.

### 1.1 Priority Settings

Users can configure the global priority for logo resolution, which can be overridden on a per-playlist basis.

**Key Strings:**
* `u_res_0x7f130316` = **"Logos priority"**
* `u_res_0x7f13031a` = **"Prefer logos from playlist"** (e.g., `#EXTINF tvg-logo="..."`)
* `u_res_0x7f130317` = **"Prefer logos from EPG"** (e.g., `<icon src="..." />` in XMLTV)
* `u_res_0x7f130318` = **"Prefer logos from folder"** (Local storage)

### 1.2 Database Implementation

The priority is stored in the `playlists` table:
* `logos_priority` (INT) — Stores the user's preferred resolution order.

During the aggregate UI query for channel rendering, TiViMate fetches both sources simultaneously to allow dynamic fallback if the preferred source is missing or fails to load.

**Source Evidence** (`C3210.java` line 44, Channel Query):
```sql
SELECT m.id, m.channel_id, c.name channel_name, c.custom_name custom_channel_name,
length(c.logo) channel_logo_from_playlist_length, length(t.icon_url) channel_logo_from_tvg_length,
p.logos_priority channel_logos_priority, ...
FROM my_programs m
LEFT OUTER JOIN channels c ON m.channel_id == c.id
LEFT OUTER JOIN playlists p ON c.playlist_id == p.id
LEFT OUTER JOIN channel_tvg_bindings b ON c.id == b.channel_id
LEFT OUTER JOIN tvg_channels t ON b.tvg_channel_id == t.id
```

Notice the query doesn't pull the full BLOB for `c.logo`. It pulls `length(c.logo)` and `length(t.icon_url)`. This determines if the logo exists before attempting to load it, saving massive amounts of memory during list scrolling.

---

## 2. LOCAL LOGO FOLDERS & INEXACT MATCHING

Users can provide a local folder of images (e.g., on a USB drive or SMB share) to use as channel logos. This is particularly useful for providers with missing or low-quality icons.

**Key Strings:**
* `u_res_0x7f130313` = **"Logos folder"**
* `u_res_0x7f130314` = **"Enter path of a folder with logos or select it"**

### 2.1 The "Inexact Matching" Algorithm

Matching local image files (e.g., `BBC One.png`) to M3U channel names (e.g., `|UK| BBC ONE HD`) is notoriously difficult due to prefixes, suffixes, and casing.

**Key String:**
* `u_res_0x7f1302e9` = **"Inexact matching for logos files"**

When this setting is enabled, TiViMate applies a fuzzy matching heuristic:
1. **Normalization:** Both the channel name and the filename are converted to lowercase.
2. **Sanitization:** Special characters (e.g., `|`, `-`, `HD`, `FHD`) and common tags are stripped or ignored.
3. **Substring Matching:** It checks if the sanitized filename is a prominent substring of the sanitized channel name, or vice versa.

---

## 3. GLIDE CACHING ENGINE

TiViMate relies on **Glide** (a popular Android image loading framework) to handle the downloading, caching, and rendering of network-based logos.

### 3.1 Caching Strategy

Glide automatically handles:
* **Memory Caching:** Keeps recently viewed logos in RAM for instant rendering while scrolling the EPG or channel list.
* **Disk Caching:** Saves downloaded logos to the device's internal storage (`cache` directory) so they persist between app restarts.

**Evidence of Glide:**
In `C1233.java` (WorkManager task loading) and `C3210.java` (Job spec parsing), we see calls to `com.bumptech.glide.ʽ.ﾞᴵ(blob)`. This confirms Glide is deeply integrated into the app's byte-loading pipeline.

### 3.2 Cache Invalidation

Because IPTV providers frequently change logos, the disk cache can become stale or excessively large.

**Key Strings:**
* `u_res_0x7f130287` = **"Clear logos cache"**
* `u_res_0x7f130312` = **"Logos cache is cleared"**

When the user triggers this action, TiViMate invokes Glide's cache clearing mechanisms:
* `Glide.get(context).clearMemory()` (Must be called on the main thread)
* `Glide.get(context).clearDiskCache()` (Must be called on a background thread)

This forces the app to re-fetch all `tvg-logo` and XMLTV `<icon>` URLs on their next appearance.
