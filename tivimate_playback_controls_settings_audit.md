# TiViMate Playback Controls & Settings Engine — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s aspect-ratio zooming layout matrices, Media3 track overrides, audio synchronization offsets, and settings panel navigation**. It dissects how the application customizes the media rendering layout on-the-fly and implements its premium slide-in TV settings drawers.

---

## 1. ASPECT RATIO ZOOMING LAYOUT MATRICES

Standard TV applications try to alter video aspect ratios by restarting hardware decoders or re-instantiating the rendering surface. This results in stutters and layout pops.

TiViMate handles format scaling (Zoom, Fit, Fill, Stretch) **purely at the Android View layout layer**, leaving hardware video decoders completely untouched.

```
                  [ Raw Stream Aspect Ratio (e.g. 4:3) ]
                                    │
                                    ▼ (Direct SurfaceView Bind)
                    [ SurfaceView Layout Container ]
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
      [ Fit Mode ]             [ Fill Mode ]          [ Stretch Mode ]
  Scale boundaries to      Crop excess; expand      Force bounds to
  fit screen, keeping      layout boundaries to     match parent 16:9,
  letterbox borders.       fully cover screen.      ignoring aspect.
```

### The Layout-Scale Override Technique
Instead of altering stream buffers, TiViMate overrides `onMeasure` inside its custom `SurfaceView` container layout. By dynamically modifying the layout bounds (`layoutParams.width` and `layoutParams.height`) relative to the screen dimensions and the stream's aspect ratio, the video scaling adjusts instantly:

*   **Normal / Fit**: Calculate dimensions keeping the exact stream aspect ratio bounded within the screen limits.
*   **Fill**: Scale the surface width and height until the shorter dimension fully matches the screen boundary, cropping excess video.
*   **Stretch**: Force `layoutParams.width = match_parent` and `layoutParams.height = match_parent` to stretch the 4:3 stream to 16:9.
*   **Zoom**: Scale up both dimensions proportionally by a user-configured scaling float (e.g., 1.2x).

---

## 2. SEAMLESS MEDIA3 MULTI-TRACK SELECTION

IPTV streams frequently contain multiple audio language channels and subtitle formats (e.g., SRT, WebVTT, CEA-608).

TiViMate manages track selection using ExoPlayer's built-in **`TrackSelectionParameters`** and track overrides. Swapping audio streams takes place instantly without restarting the decoder pipeline or reloading the media source.

```kotlin
// Replicated safe Kotlin Track Selection Override
fun selectAudioTrack(player: ExoPlayer, trackGroup: TrackGroup, trackIndex: Int) {
    val currentParameters = player.trackSelectionParameters
    
    // Build a new selection parameter that overrides active tracks
    val newParameters = currentParameters.buildUpon()
        .setOverrideForType(
            TrackSelectionOverride(trackGroup, trackIndex)
        )
        .build()
        
    // Apply parameters on-the-fly (Zero stream reload or codec restart)
    player.trackSelectionParameters = newParameters
}
```

---

## 3. AUDIO SYNCHRONIZATION CORRECTION

Often, live IPTV streams suffer from audio-to-video timestamp drift, making dialog look out-of-sync. TiViMate features an active manual **Audio Delay Slider** (allowing offsets of +/- 2,000ms).

### Under the Hood Implementation
To adjust audio delays dynamically without halting playback, TiViMate utilizes the underlying ExoPlayer **`DefaultAudioSink`** and custom audio renderers. It shifts the playback timestamps on-the-fly by modifying the audio playhead offsets in the audio sink buffer before feeding packets to the hardware audio track, guaranteeing perfect timing corrections immediately.

---

## 4. SIDE-SLIDING SETTINGS DRAWER & D-PAD NAVIGATION

Traditional fullscreen Android preference screens feel terrible on TVs because they hide the active video stream, breaking the visual immersion.

TiViMate solves this through a **Slide-in Settings Panel** (`VerticalGridView` preference drawer).

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│                                                        │
│                  Active Video Stream                   │
│                                                        │
│                                                        │
│                                      ┌─────────────────┤
│                                      │ Settings Drawer │
│                                      │ > Playlists     │
│                                      │ > EPG           │
│                                      │ > Appearance    │
└────────────────────────────────────────────────────────┘
```

### The Slide-in Navigation Hierarchy

1.  **Overlay Container Layout**:
    The main settings panel is structured inside a sliding drawer view (usually a `FrameLayout` or `RelativeLayout` with a width of 350dp) aligned to the right edge of the screen.
2.  **Leanback `VerticalGridView`**:
    Instead of standard ListViews, categories are bound inside a specialized Leanback focus grid (`VerticalGridView`).
3.  **Horizontal Slide Transitions**:
    When a category is clicked:
    *   The active side panel slides left or fades out.
    *   The new nested settings drawer slides in from the right edge with a smooth **200ms translation animation** (`translationX`).
    *   Focus is automatically routed to the top item of the new list using `.requestFocus()`, ensuring the D-pad remote flow never gets stuck.
    *   **Standby Playback**: The rest of the screen remains transparent, letting the user watch and hear their TV channel while configuring settings in real-time.
