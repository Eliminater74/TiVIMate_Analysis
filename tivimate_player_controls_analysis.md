# TiViMate Reference Analysis — Custom Player Controls & Track Selection

This document provides a highly detailed architectural analysis of **TiViMate’s** custom player controls (OSD overlays) and media track selection systems.

---

## 1. USER-FLOW LEVEL BEHAVIOR
* **Unified OSD Overlay (Quick Actions)**: Clicking D-pad Center or Down during video streaming displays a transparent dark card at the bottom. This panel overlays the active video and hosts quick actions like play/pause, subtitles, audio track selector, aspect ratio, and picture-in-picture.
* **Instant Media Track Swap**: Tapping the "Audio" or "Subtitles" buttons displays a clean modal listing the available language streams. When selected, the audio switches instantly with zero video stutter or decoder restarts.
* **Aspect Ratio / Format Shifts**: Tapping the zoom button cycles through scaling modes (Normal, Fit, Zoom, Stretch) with immediate layout resize feedback.

---

## 2. ARCHITECTURAL PATTERNS under the hood
* **Leanback OSD View Binding**: TiViMate wraps ExoPlayer's track selection logic and binds it directly to customized Android Leanback-centric UI adapters.
* **Decoupled Track Selection Controller**: The OSD controls communicate with ExoPlayer using `TrackSelectionParameters`. When a user toggles an audio or subtitle language, the selection parameters are modified, forcing ExoPlayer's `DefaultTrackSelector` to seek new tracks without rebuilding or restarting the active media codec.
* **Hardware Aspect Ratio Scaling**: Instead of modifying the raw video frame buffers, the aspect ratio zoom is handled purely at the Android View layout layer by adjusting the layout bounds (`View.layoutParams`) of the `SurfaceView` container.

---

## 3. CLUE SOURCE (CLASSES & FILES)
* **`sources/androidx/leanback/widget/ViewOnClickListenerC0083.java`** (The OSD Track Selection Controller):
  * Imports and binds to Media3's `TrackSelectionView`.
  * Intercepts clicks on the track selector rows (`ViewOnClickListenerC0083.java` line 141) and resolves them dynamically.
  * Adjusts active selection formats (`trackSelectionView.f1326 = true`) based on click parameters.
  * Coordinates track parameters dynamically:
    ```java
    TrackSelectionView trackSelectionView = (TrackSelectionView) this.f841;
    HashMap map = trackSelectionView.f1321;
    ```
* **`sources/androidx/media3/ui/TrackSelectionView.java`** (ExoPlayer track layout view):
  * Displays audio/subtitle languages and qualities. Binds to `DefaultTrackSelector` parameters dynamically to reflect currently active media streams.

---

## 4. CONCEPTUAL REPLICATIONS FOR PUREFUSIONIPTV
* **ExoPlayer TrackSelectionView Wrapper**: Integrate ExoPlayer's built-in `TrackSelectionParameters` and track selectors to manage audio and subtitle choices, wrapping it inside a Leanback-friendly TV dialog layout.
* **No Codec Restarts for Track Swapping**: Do not stop the player or reload the stream when switching audio or subtitle tracks. Simply apply new track selection parameters on the existing player:
  ```kotlin
  player.trackSelectionParameters = player.trackSelectionParameters
      .buildUpon()
      .setOverrideForType(TrackSelectionOverride(trackGroup, trackIndex))
      .build()
  ```
* **Aspect Ratio SurfaceView Scaling**: Handle format scaling (Zoom, Fit, Stretch) by updating the width and height parameters of your `SurfaceView` layout relative to the video size aspect ratio, keeping hardware video decoders untouched.

---

## 5. DIRECT COPYING WARNINGS (DO NOT COPY)
* **Leanback Default UI Classes**: Avoid copying class names or structures like `ViewOnClickListenerC0083`. This represents a decompiled click listener class that was heavily customized by Leanback or the R8 compiler to bind fields.
* **Hardcoded Track View IDs**: Do not reference hardcoded fields like `trackSelectionView.f1315`. These represent obfuscated view variables that will not match your modern standard compile-time R layout IDs.

---

## 6. IMPLEMENTATION RISKS
* **Audio Track Switching Lag**: On older devices, switching audio tracks (e.g. HLS multi-audio playlists) can cause a 1-2 second audio silence if the audio decoder needs to re-sync timestamps. Mitigate this by pre-warming media codecs or using clean ExoPlayer chunk loaders.
* **Subtitle Rendering ANRs**: Displaying complex SSA/ASS text subtitles can block the main UI thread during heavy scrolling if rendering is handled directly on standard text layers. Always offload text layout calculations to ExoPlayer's dedicated background text renderer.
