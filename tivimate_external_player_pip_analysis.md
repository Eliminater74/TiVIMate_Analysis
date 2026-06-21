# TiViMate External Player Integration & PiP Architecture

This document provides a detailed technical analysis of how TiViMate handles offloading video playback to external media players (like VLC or MX Player) and its Picture-in-Picture (PiP) implementation. It is based on string resource analysis and Android framework interactions found in the decompiled APK.

---

## 1. EXTERNAL PLAYER INTEGRATION

TiViMate offers users the choice between its internal player (ExoPlayer) and third-party external players.

### 1.1 Triggering External Playback

External playback is triggered either via a global setting or manually per-stream.

**Key Strings:**
* `u_res_0x7f1303e4` = **"Use external player"** (Global toggle setting)
* `u_res_0x7f13023e` = **"Open in external player"** (Quick action from player OSD)

### 1.2 Android Intent Mechanism

When an external player is requested, TiViMate delegates the playback responsibility to the Android OS using an implicit `Intent`.

The flow is as follows:
1. **Fetch Stream URL:** The internal URL resolver extracts the final streaming URL (M3U8, TS, MKV, etc.).
2. **Build Intent:** TiViMate constructs an `Intent` with the `ACTION_VIEW` action.
3. **Set Data and Type:** It sets the URI to the stream URL and specifies the MIME type (e.g., `video/*` or specifically `application/x-mpegURL` for HLS).
4. **Launch:** `context.startActivity(intent)` is called.

### 1.3 Format Support and Fallbacks

Because external players rely on Intent resolution, if no app on the device registers to handle the specific MIME type or protocol, the action fails.

**Key String:**
* `u_res_0x7f130411` = **"External players not found for this format"**

This error is shown when `PackageManager.resolveActivity()` returns null, indicating the OS has no suitable app to play the stream.

### 1.4 Limitations of External Players

When an external player is used, TiViMate loses control over the playback experience. Consequently:
* **No Custom OSD:** The TiViMate Info Panel and Quick Actions are gone.
* **No EPG Overlay:** The timeline and EPG data cannot be shown over the video.
* **No Timeshift/Catch-up Support:** Because TiViMate relies on intercepting seek events to dynamically rebuild URLs for catch-up (see `tivimate_catchup_controls_architecture.md`), this functionality breaks in external players.
* **No Watch Time Tracking:** TiViMate cannot track `resume_position_ms` or `last_turn_on_time` accurately.

---

## 2. PICTURE-IN-PICTURE (PiP) MODE

TiViMate fully supports Android TV's native Picture-in-Picture feature, allowing users to watch a stream in a small corner window while navigating the Android OS or other apps.

### 2.1 Triggering PiP

**Key Strings:**
* `u_res_0x7f1300f2` = **"Enter Picture In Picture Mode"** (Manual trigger via Quick Actions)
* `u_res_0x7f1301b5` = **"Picture-in-picture"** (Setting category)
* `u_res_0x7f1303c0` = **"Switch to picture-in-picture mode on press Home"** (Auto-trigger setting)

### 2.2 Auto-PiP on Home Button

The "Switch to picture-in-picture mode on press Home" feature is a major UX enhancement.

When enabled, TiViMate overrides the standard `onUserLeaveHint()` Activity lifecycle callback.

**Standard Flow (Disabled):**
`User presses Home` -> `onPause()` -> `onStop()` -> Playback stops, app goes to background.

**PiP Flow (Enabled):**
`User presses Home` -> `onUserLeaveHint()` triggered -> TiViMate calls `enterPictureInPictureMode()` -> Playback continues in PiP window.

### 2.3 PiP Lifecycle and UI Adaptation

When the app enters PiP mode, the Activity lifecycle callback `onPictureInPictureModeChanged(boolean isInPictureInPictureMode)` is triggered.

TiViMate uses this to adapt the UI:
1. **Hide Overlays:** All OSD elements, settings panels, channel lists, and EPGs are immediately hidden. Only the raw video `SurfaceView` remains visible.
2. **Aspect Ratio:** The PiP window aspect ratio is defined by the `PictureInPictureParams`. TiViMate likely sets this to match the current video's aspect ratio (e.g., 16:9).
3. **Restore:** When the user taps the PiP window to restore fullscreen, `onPictureInPictureModeChanged(false)` is fired, and TiViMate restores the capability to show overlays.
