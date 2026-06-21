# PureFusionIPTV — EPG Channel Matching & Local Broadcast Resolution Report

This document compiles the research findings on **TiViMate's EPG matching behavior** and compares it against **PureFusionIPTV's current implementation**. It identifies the exact failure points behind the local broadcast channel guide issue ("No program information") and outlines a PureFusion-safe implementation plan.

---

## A. TiViMate Observed Behavior Summary

TiViMate uses a robust, layered EPG resolution pipeline to link M3U / Xtream Codes / Stalker portal playlist channels to EPG XMLTV channels. Based on decompiled database operations (`C3210.java`) and system behaviors, the matching priority is ordered as follows:

1. **Manual EPG Assignment (Highest Priority)**
   * **Behavior**: The user can override any automatic match by manually linking a playlist channel to an EPG channel.
   * **Persistence**: This is recorded in the `channel_tvg_bindings` junction table mapping `channel_id` to `tvg_channel_id` (XMLTV channel record).
   * **Durability**: Because it is decoupled in a junction table, manual bindings survive playlist updates, group visibility changes, and app restarts.

2. **Strict tvg-id/tvg-name Matching**
   * **Behavior**: If the playlist specifies `tvg-id` or `tvg-name` matching an XMLTV channel ID (`<channel id="...">`) exactly, the app maps it.

3. **Strict Display Name Match**
   * **Behavior**: If the normalized display name of the channel matches the XMLTV `<display-name>` or XMLTV ID exactly.
   * **Normalization**: The normalizer strips special characters, collapses repeated whitespace, converts to lowercase, and strips trailing quality tags (HD, SD, FHD, 4K, HEVC).

4. **Fuzzy Callsign / Local Broadcast Match**
   * **Behavior**: The matcher handles local network + callsign pairs (e.g. `NBC WFLA`, `ABC WFTS`, `FOX WTVT`). If the playlist channel has a local callsign token (e.g., starting with K or W, with 3-4 letters) in its name, it automatically aligns with the regional XMLTV channel ID (like `NBCWFLA.us`).
   * **Branding & Logo Matches**: If names are ambiguous, the matcher correlates logo file names and TVG logo URLs to resolve regional stations.

5. **Cross-Source Strong Fallback**
   * **Behavior**: If no program information is resolved within the channel's own playlist scope, TiViMate falls back to matching against external EPG sources (cross-source matching), but only when a strong identity (like a callsign or tvg-id) is present to prevent false matches.

6. **EPG Time Shift Layers**
   * **Behavior**: Supports layered shifts resolved in this priority:
     1. **Per-Channel Shift**: Individual channel offset settings.
     2. **Playlist Shift**: Default offset for all channels in the playlist.
     3. **EPG Source Shift**: Default offset for the XMLTV provider.
     4. **Global Offset**: General settings fallback.

---

## B. PureFusion Current Behavior Summary

PureFusion implements EPG matching using `EpgChannelMatchKeyPolicy.kt` and preloads EPG data in `EpgCacheManager.kt` using structured lookup keys.

1. **Matching Key Generation**:
   * Uses `EpgChannelMatchKeyPolicy.queryKeys(channel)` to generate candidate lookup strings.
   * If a channel is named `"WFLA"`, it parses it as a call sign and adds variants for all standard local networks (e.g., `"nbcwfla.us"`, `"abcwfla.us"`, `"wfla"`).
   * If a channel is named `"NBC WFLA"`, it resolves to a specific network/callsign pair and generates specific keys: `"nbcwfla"`, `"nbc wfla"`, `"wfla"`, `"nbcwfla.us"`.
   * If a channel is named `"NBC"`, it is considered a generic name and only generates `"nbc"`.

2. **Active XMLTV Parsing Filter**:
   * When syncing EPG, `EpgManager` calls `parseXmltvStreamingWithBatchInsert` passing a `XmltvActiveChannelFilter` loaded via `XmltvActiveChannelFilter.load(channelDao, playlistId)`.
   * The filter builds `activeIds` from active channels using `queryKeys`.
   * For each `<programme>`, it evaluates `isMapped(xmltvChannelId)` by checking if any variant of the EPG ID exists in `activeIds`. If not, it skips parsing the programme elements entirely to save memory and CPU.

3. **EPG Cache Lookup**:
   * `EpgCacheManager.preloadProgramsForGroup` loads EPG programs from the database in chunks.
   * It queries `epgManager.getProgramsInTimeRangeList` passing `playlistIds` to scope the database query.
   * Scoped queries filter by `(playlistId IS NULL OR playlistId IN (:playlistIds))` to isolate EPG programs to the channel's playlist.

4. **Cross-Source Fallback**:
   * If a channel has no programs loaded, `crossSourceStrongFallback` checks if the channel has a strong identity (requires a `tvg-id`, `tvg-name`, or callsign).
   * It queries the DB globally and runs `EpgCrossSourceFallbackPolicy.selectFallbackPrograms` to borrow programs from foreign playlists.

---

## C. Exact Likely Failure Points for Local Broadcast "No Program Information"

Through analyzing both the PureFusion matching engine and XMLTV indexing pipeline, the following three core failure points were identified:

### 1. Active XMLTV Filter Skips Local Channels due to Brand-only Playlist Names
If the playlist channel is named generically (e.g., `"NBC"`, `"FOX"`, or `"FOX 13"`) and has no `tvg-id`/`tvg-name` associated with it:
* `EpgChannelMatchKeyPolicy.queryKeys` generates only `"nbc"` or `"fox"` (avoiding callsign variants to prevent false positives).
* During XMLTV parsing, the EPG channel ID is `"NBCWFLA.us"` (or `"FOXWTVT.us"`). Its query keys are `"wfla"`, `"nbcwfla"`, etc.
* Since `"nbc"` is not in the keys for `"NBCWFLA.us"`, `XmltvActiveChannelFilter.isMapped("NBCWFLA.us")` returns `false`.
* **The parser skips the programme tags for `NBCWFLA.us` completely.** They are never written to the database.
* Consequently, even if a user manually maps `"NBC"` to `"NBCWFLA.us"` in the UI, the database contains **zero program rows** for `"NBCWFLA.us"`, resulting in "No program information".

### 2. Playlist Scoping Isolation in EPG Preloading
In `EpgCacheManager.preloadProgramsForGroup` (line 386), the DB query is restricted to `playlistIds` (the playlist ID of the active channel group).
* If EPG data is parsed and stored under `playlistId = X` (playlist X's sync), but the active channel belongs to playlist `Y`:
* The strict preloader query `getProgramsInTimeRangeListScoped` filters by `playlistId IN (Y)`.
* It returns 0 programs because the data is tagged with `playlistId = X`.
* The programs never enter the RAM lookup cache, resulting in a blank timeline.

### 3. Cross-Source Fallback Filtering Bug
If the preloader fails to load the programs, `crossSourceStrongFallback` is triggered.
* It fetches candidate EpgPrograms globally (where `playlistId = X` is returned).
* It then calls `EpgCrossSourceFallbackPolicy.selectFallbackPrograms(channel, candidates, ownPlaylistIds)`.
* If the channel is in playlist `Y`, and we have candidates synced under `playlistId = Y` (but not cached due to match keys discrepancy), `selectFallbackPrograms` filters them out because:
  `program.playlistId !in ownIds` -> `Y !in [Y]` is `false`!
* Thus, the fallback policy actively rejects programs belonging to the channel's own playlist, assuming the strict path already loaded them. Since the strict path didn't, the channel is left with no data.

---

## D. PureFusion-Safe Implementation Plan

To fix these issues safely without copying TiViMate code, modifying obfuscated classes, or introducing false-positive matches (like `ABC` matching `ABC News`), we will implement the following strategies:

### 1. Multi-DisplayName Mapping in XMLTV Ingestion
Modify the XMLTV parser in `EpgManager.kt` to parse `<display-name>` tags inside the `<channel>` block.
* When parsing `<channel id="NBCWFLA.us">`, read all nested `<display-name>` text elements (e.g. `NBC WFLA`, `WFLA`, `NBC`).
* Map these display names to the EPG channel ID in a temporary in-memory map or index.
* When evaluating `XmltvActiveChannelFilter.isMapped(xmltvChannelId)`, check if the active channel names (e.g. `"NBC"`) match any of the display names declared for `"NBCWFLA.us"` in the XMLTV file.
* This allows generic names to safely align with local callsign IDs *only when the XMLTV source explicitly lists them as aliases*, preventing arbitrary false-positive mappings.

### 2. Loose Local Broadcast Resolution Guard
Update `EpgChannelMatchKeyPolicy.kt` to support clean branding resolution:
* If a playlist channel name consists of a local network followed by numbers/branding (e.g., `"FOX 13"`), and the XMLTV EPG ID contains that network and a valid callsign (e.g. `"FOXWTVT.us"`), allow a "weak local match" *if and only if* it is the only local broadcast channel for that network in the current EPG source.
* Add specialized brand-suffix exclusions. If an XMLTV ID or channel name contains words like `"News"`, `"Sports"`, `"Now"`, `"Network"`, `"Weather"`, or `"Business"`, they must match exactly, protecting against `"ABC"` matching `"ABC News"`.

### 3. Global/Shared EPG Scoping Fallback
Adjust the preloader query in `EpgCacheManager.kt`:
* If an EPG source is shared across multiple playlists, EpgPrograms should either be written with `playlistId = null` (global) or `EpgCacheManager` must query EPG programs across all playlist scopes and then let the match key matcher select the correct row.

---

## E. Recommended Files/Classes to Modify Later

1. [EpgManager.kt](file:///I:/GITHUB/Projects/PureFusionIPTV/app/src/main/java/dev/eliminater/purefusioniptv/epg/EpgManager.kt)
   * **Changes**: Update `parseXmltvStreamingWithBatchInsert` to capture `<channel>`'s `<display-name>` tags and pass them to the active channel filter.
2. [XmltvActiveChannelIndex.kt](file:///I:/GITHUB/Projects/PureFusionIPTV/app/src/main/java/dev/eliminater/purefusioniptv/epg/XmltvActiveChannelIndex.kt)
   * **Changes**: Extend `XmltvActiveChannelFilter` to support alias-matching based on XMLTV channel display names.
3. [EpgChannelMatchKeyPolicy.kt](file:///I:/GITHUB/Projects/PureFusionIPTV/app/src/main/java/dev/eliminater/purefusioniptv/epg/EpgChannelMatchKeyPolicy.kt)
   * **Changes**: Refine local broadcast key generation and add brand-suffix exclusion guards.
4. [EpgCacheManager.kt](file:///I:/GITHUB/Projects/PureFusionIPTV/app/src/main/java/dev/eliminater/purefusioniptv/epg/EpgCacheManager.kt)
   * **Changes**: Update preloading database queries and caches to resolve shared playlist EPG rows.

---

## F. Tests to Add

1. **XMLTV Display Name Mapping Tests**
   * Verify that when XMLTV has `<channel id="NBCWFLA.us"><display-name>NBC</display-name></channel>`, the filter does not skip `"NBCWFLA.us"` programs when the playlist contains a channel named `"NBC"`.
2. **Branding Suffix Guard Tests**
   * Assert that `"NBC"` does NOT match `"NBC News Now"` or `"NBC News"`.
   * Assert that `"FOX"` does NOT match `"Fox Sports"`.
   * Assert that `"ABC"` does NOT match `"ABC News"`.
3. **Cross-Playlist Shared EPG Cache Tests**
   * Verify that when EPG is synced under playlist `1`, a channel in playlist `2` with matching EPG credentials can successfully load EPG rows from the cache.

---

## G. Risks and False-Positive Guards

* **Risk: Out-Of-Memory (OOM) from Display-Name caching**
  * *Mitigation*: The in-memory XMLTV channel-to-display-name map must only store active mappings or be cleared immediately after EPG parsing concludes.
* **Risk: Over-eager matching**
  * *Mitigation*: Enforce strict token-equality on normalized strings. Substring checks (like `contains()`) should never be used on generic network prefixes without callsigns.
* **Risk: DB query performance regression**
  * *Mitigation*: Ensure all new scoped query variants utilize existing index structures (`channelEpgId`, `startTime`, `endTime`, `playlistId`).
