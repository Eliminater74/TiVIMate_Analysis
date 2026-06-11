# TiViMate Channel & Group Customization Architecture — Reference Analysis

This document provides a highly detailed architectural analysis of **TiViMate’s channel and category customization subsystem**. It explains how the app implements user-defined custom groups, manages row-action reordering, and handles parental blocks dynamically at the database and UI layout layer.

---

## 1. CUSTOM GROUPS & CHANNEL MAPPING ARCHITECTURE

IPTV providers offer massive, cluttered channel lists. A premium app must let users create **custom groups** (e.g., "Kids Channels," "Sports Pack") and copy/move streams into them across different playlists.

```
 [ Provider Playlist A ] ────► [ Channel Row (ID: 104) ]
                                      │
                         (Map to virtual custom group)
                                      ▼
                       [ group_channel_mappings ]
                       - custom_group_id = 902
                       - channel_id = 104
                       - position = 1
                                      ▲
                         (Query Virtual Category)
                                      │
 [ User Custom Group (ID: 902) ] ─────┘
```

### The Virtual Mapping Pattern
To support custom categories without duplicating heavy streaming rows or EPG structures in SQLite, TiViMate leverages a **Junction Mapping Table** (`group_channel_mappings`):

*   **`custom_groups`**: Holds user-created category details (`id`, `playlist_id` [usually a master virtual playlist ID], `name`, `position`, `is_visible`).
*   **`group_channel_mappings`**: Connects channels directly to these virtual folders.
    *   Columns: `group_id`, `channel_id`, `position_in_group`.
    *   *Why this matters*: A single channel row can reside in its original provider group, but also exist inside multiple custom user folders seamlessly. This keeps SQLite normalized and saves storage.

---

## 2. COPYING, MOVING, AND REORDERING CHANNELS

*   **Row Actions**: In the settings screen (Manage Groups / Manage Channels), the user is presented with a vertical row of actions: "Move Up," "Move Down," "Copy to Group," "Move to Group," "Hide," or "Lock."
*   **Move Operation**:
    Updates the position column inside the mapping table, triggering an index shift block in SQLite.
*   **Copy Operation**:
    Inserts a new relationship row into `group_channel_mappings` with a fresh `position_in_group` index, keeping the primary channel metadata untouched.

---

## 3. PARENTAL LOCK GROUP INTEGRATION

To restrict access to adult content or specific categories, users can apply a **Parental Lock** on groups or individual streams.

### The Locking Mechanics
*   **Database Flag**: The `categories` and `channels` tables contain an `is_locked` integer flag (`0` or `1`).
*   **UI Selection Interception**:
    When the user navigates the EPG or channel list and highlights a locked item, the UI does not immediately bind the stream URL. Instead:
    1.  The controller checks: `if (channel.isLocked == 1)`.
    2.  If true, it halts focus navigation, overlays a **secure PIN prompt card**, and caches the target channel ID in memory.
    3.  If the correct PIN is entered, the app unlocks playback and binds the stream.
    4.  **Session Timeout**: The lock state remains "temporarily unlocked" during the active viewing session. If the user zaps away or stays inactive for 30 minutes, the session expires, and the channel relocks automatically.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Add a nullable `is_locked` integer flag to your `ChannelEntity` and `CategoryEntity` Room classes.
*   Filter out locked streams from standard EPG and VOD search queries unless parental control is unlocked globally in the app session.

### Medium-Risk Actions
*   Implement custom groups using a separate junction table (`group_channel_mappings`) to prevent duplicate channel records.
*   Design a dedicated slide-in "Manage Channels" UI settings panel to reorder channel positions dynamically.

### High-Risk Actions
*   Physically deleting channel rows when moving them between groups. Always use mapping table modifications to prevent database constraint exceptions and preserve user configuration states.

### What to Measure First
*   Benchmark dynamic junction joins. Resolving custom group contents must finish in **under 2ms** to maintain a responsive D-pad scrolling experience.

### What Not to Copy Directly
*   Do not copy custom layout wrappers or reordering adapters from `p312.AbstractC3857`.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Secure soft parental locks and a robust group mapping table are core requirements for playlist customization.
