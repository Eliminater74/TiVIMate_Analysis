# TiViMate Catch-up, PVR & Reminders — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s EPG-integrated catch-up playback, background recording scheduler, and program reminders engine**. It explains how the app interpolates timeshift streaming tokens, schedules exact wake alarms, and handles background coordination conflicts.

---

## 1. CATCH-UP & ARCHIVE PLAYBACK ENGINE

IPTV catch-up (timeshift) allows users to watch past EPG broadcasts. Since servers store this content as dynamic files indexed by timestamps, TiViMate translates timeline navigation into dynamic HTTP queries.

```
 [ EPG Guide: Click past program ] ──► [ Get program start & stop timestamps ]
                                                       │
                           (Interpolate Timeshift Streaming URL)
                                                       ▼
 [ Build URL via CatchupType ] ◄───────────────────────┴───────────────────────► [ Build URL via CatchupSource ]
 - Xtream: Inject /timeshift/ and formatted time.                              - Replace: ${start}, ${duration}
 - Flussonic: Append -{start}-{duration}.ts                                      and timezone offsets dynamically.
```

### The EPG Integration Flow

1.  **Playability Verification**:
    The EPG grid highlights playable past programs with a small "clock" icon. The UI determines playability by validating:
    `if (program.stopSeconds < currentTimeSeconds && currentPlaylist.catchupDays > 0)`
2.  **Dynamic URL Resolving**:
    When a past program is selected, the player controller fetches the channel's `catchup_type` and `catchup_source` parameters. It maps timestamps on-the-fly:
    *   **Xtream Codes**: Re-formats path segments: `/timeshift/{user}/{pass}/{duration}/{start_time_formatted}/{stream_id}.ts`
    *   **Flussonic**: Appends stream parameters: `/{stream_name}/mono-timeshift-{start_seconds}-{duration}.ts`
    *   **Append**: Tokenizes strings containing `${start}` and `${duration}` variables.
3.  **Timeshift Seeking Mapping**:
    ExoPlayer binds the resolved timeshift URL. Dragging the progress slider translates the seek position directly into absolute Epoch Times, updating the URL query parameters dynamically and reloading the player without restarting visual decoding.

---

## 2. ADVANCED PVR / DVR BACKGROUND SCHEDULING

To record live video stream feeds without active interface overhead, TiViMate runs a database-backed background pipeline.

### The Storage & Networking Architecture
*   **Storage Target Decoupling**:
    Recordings can be directed to local storage (internal flash or USB drives via SAF) or remote network shares (SMBv2/v3).
*   **Directory Mapping**:
    Active writes are written to a temp folder: `/cache/pvr/temp_rec_{id}.ts`. Once the schedule concludes safely, the file is moved to its permanent target folder and renamed with EPG meta tags: `ChannelName - ProgramTitle - Date.ts`.
*   **Multi-Stream Recording limits**:
    If the IPTV provider allows multiple active connections, multiple `PvrForegroundService` tasks can run concurrently. If a conflict occurs (connection limit reached), the newer scheduled recording is marked `FAILED_CONNECTION_LIMIT` in SQLite, protecting the older active recording from corruption.

---

## 3. PROGRAM REMINDERS & ALARMS ENGINE

When browsing the EPG, users can schedule **Reminders** for future programs.

### The Scheduling & Conflict Routine
*   **Exact Alarm Scheduling**:
    Reminders are registered using Android's `AlarmManager` exact triggers. They persist across device reboots by hooking into `BOOT_COMPLETED` receivers.
*   **Overlay Notification Display**:
    When the alarm triggers, a lightweight, transparent system overlay card pops over the active video playback:
    ```
    ┌────────────────────────────────────────────────────────┐
    │  Reminder: "Sunday Night Football" is starting now!    │
    │  [ Tune Channel ]                 [ Dismiss ]          │
    └────────────────────────────────────────────────────────┘
    ```
*   **Zapping Conflict Safety**:
    If the user clicks "Tune Channel," the app checks if an active PVR recording is currently running. If the playlist only supports a single connection, the reminder engine prompts a warning dialog: *"Tuning will stop your active recording. Continue? [Yes] [No]"* to safeguard data integrity.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Design custom layouts for program reminders using simple AlarmManager wake receivers.
*   Configure EPG past timelines to highlight historical items with a standard translucent watch icon.

### Medium-Risk Actions
*   Implement dynamic URL timeshift token resolvers inside your repository player layers.
*   Build a safety conflict manager that checks active recording statuses before zapping channels on single-stream accounts.

### High-Risk Actions
*   Running foreground service recordings on API 30+ without declaring `foregroundServiceType = dataSync` or `mediaPlayback` in the Android Manifest, which results in immediate OS-level security crashes.

### What to Measure First
*   Measure network handshake times. Timeshift URLs should start streaming in **under 1.5 seconds** on target hardware.

### What Not to Copy Directly
*   Do not copy custom alarm receiver mappings from `p262.C3419`.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Secure, precise AlarmManager scheduling for PVR recording and EPG program reminders.
