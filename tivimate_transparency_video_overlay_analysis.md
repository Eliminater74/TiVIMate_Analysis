# TiViMate Transparency & Video Overlay Architecture — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s layered transparency and background video rendering system**. It explains how the application displays active live video streams smoothly behind EPG grids, search layouts, and settings drawers without dropping frames, outlining the window, view hierarchy, and alpha scrim techniques used to balance readability with cinematic immersion.

---

## 1. USER-FLOW LEVEL BEHAVIOR

*   **Cinematic Background Streaming**: When the user opens the TV Guide (EPG grid), navigates the Search panel, or enters Settings, the active video channel does not stop playing or get squashed into a tiny window. It continues rendering in full screen behind the UI controls.
*   **The Dimming Scrim (Legibility Guard)**: To make program titles, lists, and settings text easy to read over bright or fast-moving video frames (like a sports broadcast), a translucent dark layer (scrim) gently dims the background video when menus are summoned.
*   **Glassmorphic Menus**: Sidebar menus and settings drawers utilize translucent dark panels, allowing the video's colors and motions to softly bleed through the edges of the UI cards.

---

## 2. THE THREE-LAYER WINDOW VIEW HIERARCHY

To display active video behind UI overlays, Android TV applications must organize their layouts in a strict, layered depth stack (Z-index). The video surface must reside at the very bottom, and all layers on top must be configured to permit transparency.

```
  ┌────────────────────────────────────────────────────────┐
  │ Layer 2: Interactive UI Controls (EPG Grid, Search)   │  <-- Translucent HSL fills (#CC0B0F19)
  ├────────────────────────────────────────────────────────┤
  │ Layer 1: Legibility Scrim Overlay (Dynamic Alpha View) │  <-- Animate alpha 0.0f to 0.6f on menu open
  ├────────────────────────────────────────────────────────┤
  │ Layer 0: Hardware Media Player (SurfaceView / Video)   │  <-- Renders behind main window layer
  └────────────────────────────────────────────────────────┘
```

### Layer 0: The Hardware SurfaceView (Bottom)
In TiViMate, ExoPlayer binds its video output to a **`SurfaceView`** (rather than a standard `TextureView`).
*   *Why SurfaceView is selected*: `SurfaceView` renders on a dedicated hardware-composited window layer directly behind the main Android view window, bypassing the main UI thread entirely. This prevents EPG scrolling lag from stuttering the video.
*   *The Transparency Requirement*: Because `SurfaceView` sits behind the main window, **every single layout layer on top must be translucent or transparent**, otherwise the main window will draw a solid background, blocking the video surface.

### Layer 1: The Legibility Scrim (Middle)
An un-dimmed video makes overlay text unreadable. TiViMate positions a dedicated **Scrim View** immediately above the `SurfaceView`:
*   *The Scrim View*: A simple layout `View` configured with a solid dark background (`#000000` or `#0B0F19`).
*   *Dynamic Alpha Control*: When no menus are visible, the scrim's alpha is set to `0.0f` (fully invisible). When the EPG or Search overlay is opened, the scrim's alpha animates to `0.5f` or `0.6f` (50%-60% opacity) over **200ms**. This dims the video, making the overlay text highly legible while preserving video motion.

### Layer 2: The Glassmorphic UI (Top)
The EPG grid and search overlays sit at the top. They use **translucent HSL color tokens** (e.g. `#CC0B0F19` which is 80% opaque deep slate) instead of solid black or dark gray, allowing the dimmed video's colors to softly bleed through.

---

## 3. THE TRANSPARENT SYSTEM THEME OVERRIDE

If the Android Application Theme has a solid window background, the system will render a solid dark canvas behind your views, completely blocking the video surface.

TiViMate overrides this by applying a translucent theme to its `MainActivity` inside the Android Manifest:

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.TvPlayer.Translucent">
</activity>
```

The corresponding styles XML sets the window background to transparent and disables standard window overlays:

```xml
<!-- res/values/styles.xml -->
<resources>
    <style name="Theme.TvPlayer.Translucent" parent="Theme.Leanback">
        <!-- Disable solid window backgrounds -->
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:colorBackgroundCacheHint">@null</item>
        <item name="android:windowIsTranslucent">true</item>
        <!-- Prevent window dimming overlays from standard themes -->
        <item name="android:backgroundDimEnabled">false</item>
    </style>
</resources>
```

---

## 4. MULTI-TIER PERFORMANCE OPTIMIZATIONS (NO BLUR OVERHEAD)

Real-time Gaussian blur effects (e.g., using `RenderEffect` on Android 12+ or ScriptIntrinsicBlur on older devices) are incredibly expensive on GPU cycles. Running real-time blur over a 1080p 60fps video stream will immediately **crash or stutter budget TV sticks (like Firesticks)**.

TiViMate achieves its high-end glassmorphic look **without using heavy real-time blurs**:
*   **Static Alpha Colors**: It uses simple HSL colors with alpha transparency values (`#CC0B0F19`). This requires **zero GPU rendering calculations**, leaving all graphics resources free to maintain a fluid 60fps scrolling guide speed.
*   **Gradient Scrims**: Instead of a flat dark scrim, it uses vertical linear gradients that transition from deep dark slate at the bottom to transparent at the top, blending the video smoothly behind overlays.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Set your `MainActivity` theme's `android:windowBackground` to `@android:color/transparent` and `android:windowIsTranslucent` to `true` in your styles XML.
*   Use ExoPlayer's `SurfaceView` rather than `TextureView` to leverage hardware composition.

### Medium-Risk Actions
*   Implement a **Legibility Scrim View** between your player surface and UI layers. Animate the scrim's alpha from `0.0f` to `0.6f` over 200ms when menus are opened, and back to `0.0f` when closed.
*   Use semi-translucent colors (`#CC0B0F19` for 80% opacity) on your settings cards and EPG cell containers.

### High-Risk Actions
*   Integrating complex real-time Gaussian blur libraries to blur the active video stream behind overlays. This will immediately cause massive frame drops and scrolling lag on budget Android TV chipsets.

### What to Measure First
*   Benchmark EPG scrolling frame rates with the video playing in the background. The UI must sustain a steady **60fps** scrolling speed during active playback.

### What Not to Copy Directly
*   Do not copy obfuscated style names or resource parameters.
