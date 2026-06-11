# TiViMate Auxiliary Systems (Logo Matching, External Resume, and Sleep Timers) — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s auxiliary systems**. It dissects how the app implements fuzzy inexact matching for channel logos, syncs playback resume positions back from third-party players (like VLC and MX Player), and structures its standby sleep timer engine.

---

## 1. THE INEXACT FUZZY LOGO RESOLUTION CHAIN

IPTV playlists often have missing logos, dead image links, or mismatches between EPG icons and channel logos. To solve this, TiViMate implements a **Layered Logo Resolution Chain** and a **Fuzzy String Matcher** to search local storage folders.

```
 [ Query Channel Row ] ──► [ Check User Custom Assigned Logo Path ] ──► (Found -> Bind)
                                        │
                                        ▼ (Null / Missing)
             [ Resolve via logos_priority settings (Playlist Override) ]
             ├─► EPG First: Check EPG icon_url ──► Playlist tvg-logo
             └─► Playlist First: Check tvg-logo ──► EPG icon_url
                                        │
                                        ▼ (Both Dead / Null)
                 [ Search Local Directory via Fuzzy Matcher ]
                 * Strip whitespace, convert to lowercase, remove accents
                 * Scan /local/logo_folder/ for matching filenames (e.g. "HBO HD")
```

### The Inexact Fuzzy String Matcher
When the user maps a local logo directory, the app scans folder filenames. To match a stream named `"H.B.O. HD (US)"` with a file named `hbohd.png`, the app runs a string-cleansing normalizer:
*   Converts characters to lowercase.
*   Removes all punctuation and special characters (periods, parentheses, brackets, hyphens).
*   Strips all whitespaces.
```
"H.B.O. HD (US)" ──► Normalize ──► "hbohd"
"hbo_hd.png"     ──► Normalize ──► "hbohd"  --> MATCH!
```

---

## 2. EXTERNAL PLAYER RESUME PROGRESS SYNC

Many users prefer playing VOD content (4K HDR movies) in external players like **VLC** or **MX Player** due to better audio passthrough support. However, external playback usually breaks watch-history tracking.

TiViMate solves this by launching external players using **`startActivityForResult`** and capturing returned progress metrics inside `onActivityResult`.

```
 [ MainActivity ] ──► startActivityForResult(Intent, 9002) ──► [ VLC / MX Player ]
        ▲                                                               │
        │ (Returns Intent: "position" & "duration" on Back press)       │
        └───────────────────────────┴───────────────────────────────────┘
                                    │
                                    ▼ (Extract milliseconds)
               [ SQLite: UPDATE movies SET resume_position_ms = ? ]
```

### The Integration Flow
1.  **Intent Dispatching**: The app launches the external player using `startActivityForResult(intent, REQUEST_CODE)`.
2.  **Intent Headers**: It sets standard flag packages so the external player returns results upon closing:
    *   `intent.putExtra("return_result", true)`
3.  **Result Capture (`onActivityResult`)**:
    When the user exits the external player and returns to the TV app, `onActivityResult` catches the returned bundle. Standard external players (VLC, MX Player, JustPlayer) write progress metrics into the result intent:
    *   **VLC**: Returns current position in `"position"` (Long, milliseconds) and total length in `"duration"`.
    *   **MX Player**: Returns `"position"` (Int, milliseconds).
4.  **Database Commit**: The TV app extracts these milliseconds and updates the SQLite playhead resume marker (`resume_position_ms`) immediately, ensuring watch states remain perfectly synchronized.

---

## 3. STANDBY SLEEP TIMER ENGINE

Smart TVs are frequently left running overnight. TiViMate incorporates a **Sleep Timer** that cleanly shuts down active streams and displays a black screen or triggers system standby after a user-configured interval (15, 30, 60, 120 minutes).

### The Architecture
*   **The Background Scheduler**: When activated, the timer spawns a background thread-safe scheduler (`ScheduledExecutorService` or `Handler.postDelayed()`).
*   **Dynamic UI Warning**: One minute before the sleep timer concludes, the UI overlays a transparent notification card: *"Sleep timer concluding in 60 seconds. Press any button to cancel."*
*   **The Clean Closure**: If no D-pad key is pressed, the app:
    1.  Calls `player.stop()` and `player.release()`, releasing all hardware decoders.
    2.  Sets the screen brightness or view overlay to full black.
    3.  Attempts to put the TV panel to sleep using CEC standby commands, or calls `activity.finishAffinity()` to exit the process cleanly.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Build a **Fuzzy String Normalizer** in Kotlin to strip special characters and whitespace when matching EPG channel names with playlist icons.
*   Implement a persistent background Sleep Timer using simple coroutine-based delay runners or Handlers.

### Medium-Risk Actions
*   Implement the **External Player Activity-Result Listener** to capture and synchronize playhead positions back from VLC, MX Player, and JustPlayer upon closing:
    ```kotlin
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if (requestCode == REQUEST_EXTERNAL_PLAYER && data != null) {
            val positionMs = data.getLongExtra("position", -1L) // VLC format
                if (positionMs > 0) {
                    saveResumePositionToDb(activeMovieId, positionMs)
                }
        }
    }
    ```

### High-Risk Actions
*   Running inexact folder matches over thousands of logo files synchronously on the main thread, which blocks TV UI rendering. Always run fuzzy filename mappings asynchronously inside IO Coroutines.

### What to Measure First
*   Verify external player exit times. Swapping back to the TV app and writing the resume position must occur in **under 100ms** to prevent screen flickering.

### What Not to Copy Directly
*   Do not copy obfuscated Glide integration modules or internal result callbacks.

### Suggested Implementation Phase
*   **Beta 10 UX Polish (P2)**: Fuzzy local logo directories matching and external player resume synchronization are elite features for modern IPTV players.
