# TiViMate Reference Analysis — Playlist Parser Optimizations

This document provides a highly detailed architectural analysis of **TiViMate’s** M3U/M3U8 playlist parsing strategies, detailing the streaming parse loops and tokenization mechanisms.

---

## 1. USER-FLOW LEVEL BEHAVIOR
* **Instant List Loading**: When the user adds a massive playlist file (containing 10,000 to 50,000 channels, VOD movies, and TV shows), the app begins importing files instantly.
* **Low Memory Footprint**: Even on low-spec Android TV boxes with only 1.5GB of RAM, importing very large playlist files does not trigger "Out of Memory" crashes or make the app freeze during parsing.
* **Visual Progress Status**: The user is presented with a fast, real-time incrementing channel loader rather than an indefinite frozen spinner.

---

## 2. ARCHITECTURAL PATTERNS under the hood
* **Streaming Input Parsing (Tokenization Pattern)**: TiViMate completely avoids loading the raw playlist text string or file bytes into RAM at once. Instead, it processes the playlist line-by-line as a data stream using a buffered character reader.
* **Metadata Queue Buffering**: It parses lines into an ordered token queue (`ArrayDeque`) as it loops through the stream. It isolates metadata rows (like `#EXTINF` or `#EXT-X-STREAM-INF`) and resolves their parameters sequentially without creating temporary large string allocations.
* **Polymorphic Playlist Mapping**: It parses channels, video codecs, aspect ratios, and media groups in a single-pass scanner, routing live streams and VOD entries directly to their relational database models.

---

## 3. CLUE SOURCE (CLASSES & FILES)
* **`sources/p047/C1367.java`** (The core M3U/M3U8 streaming parser):
  * `mo4043(Uri uri, C3454 inputStream)`: Parses the playlist stream. Binds an active `BufferedReader` wrapped in an `InputStreamReader` to parse lines dynamically:
    ```java
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(c3454));
    ```
  * `zM7779` checks for the `#EXTM3U` header signature sequentially by reading individual characters (`bufferedReader.read()`) at the start of the stream, completely avoiding full-string allocations.
  * Loops line-by-line (`bufferedReader.readLine()`), trimming whitespace and pushing valid metadata tags (such as `#EXT-X-STREAM-INF` or `#EXTINF`) to a custom `ArrayDeque` queue.
  * Passes the token queue and reader to `m4042(new ﹳٴ(arrayDeque, bufferedReader))` to parse attributes sequentially.
  * Extracted properties (like resolutions `i6 x i5` or frame rates `f`) are mapped directly to ExoPlayer's `C1490` track format builder.

---

## 4. CONCEPTUAL REPLICATIONS FOR PUREFUSIONIPTV
* **BufferedReader Streaming**: Do not read full M3U files into a single String or memory array. Always pass the stream to a `BufferedReader` and parse lines sequentially inside a `while ((line = reader.readLine()) != null)` loop.
* **Attribute Tokenizer**: Write a lightweight parser that seek-scans strings sequentially for attributes (e.g. `tvg-id=`, `group-title=`, `tvg-logo=`) instead of calling heavy regex `Pattern.matcher` on every single channel line (which creates immense garbage collection pressure).
* **Batch Database Commits**: Rather than writing each parsed channel to SQLite one-by-one, accumulate items in a memory buffer (e.g., list of 500 items) and commit them inside unified Room database transactions to maximize I/O throughput.

---

## 5. DIRECT COPYING WARNINGS (DO NOT COPY)
* **Obfuscated Parser Utilities**: Avoid mimicking class structures or mapping values from `p047.C1367` or `p047.C1360`. These contain obfuscated layout wrappers for ExoPlayer's parsing structures.
* **ExoPlayer Internal Format Mapping**: Do not copy format bindings like `C1490 c1491 = new C1490(); c1491.f5857 = strM40314`. These represent ExoPlayer's internal decompiled format fields which vary heavily between different Media3 player versions.

---

## 6. IMPLEMENTATION RISKS
* **GC Pressure from Regular Expressions**: Using standard Java regex patterns (like `Pattern.compile(...)`) inside a loop over a 40,000-line playlist will cause severe micro-stuttering and heap exhaustion due to millions of temporary matcher allocations.
* **Malformed Playlist Crashes**: Raw M3U files often contain corrupted tags, missing quotes, or weird characters. The sequential parser must implement strict boundary checks and try-catch filters on integer and float parses (`Integer.parseInt()`) to prevent crashes during imports.
