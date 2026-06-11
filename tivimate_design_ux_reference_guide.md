# TiViMate Design, UX & Visual Reference Guide (Advice for TV App Development)

This master reference guide compiles all the high-end UX design principles, visual tokens, layout structures, and remote control (D-pad) handling behaviors extracted from **TiViMate’s** architecture. It includes standard visual parameters, dark-mode styling variables, remote control timing rules, and an embedded custom UI mockup to serve as a design blueprint.

---

## 1. THE PREMIUM "ZERO-LIGHT" VISUAL DESIGN SYSTEM

Modern Android TV apps should feel cinematic and luxurious. This is achieved by utilizing curated dark palettes (deep slate HSL coordinates) combined with semi-translucent glassmorphic panels, rather than harsh pitch-black backgrounds.

### Color Palette Specification

```xml
<!-- res/values/colors.xml -->
<resources>
    <!-- Backgrounds (Deep Dark Slate) HSL(222, 47%, 8%) -->
    <color name="bg_cinematic_dark">#0B0F19</color>
    
    <!-- Semi-Translucent Overlays (80% Opacity) -->
    <color name="bg_glass_overlay">#CC0B0F19</color>
    
    <!-- Active D-pad Focus Accent HSL(217, 91%, 60%) -->
    <color name="accent_focus_glow">#3B82F6</color>
    
    <!-- Text Elements -->
    <color name="text_primary">#F8FAFC</color>   <!-- Near-white for high legibility -->
    <color name="text_secondary">#94A3B8</color> <!-- Soft slate for timestamps/subtitles -->
    
    <!-- EPG Program Cell States -->
    <color name="bg_epg_cell">#1E293B</color>     <!-- Premium Slate Gray -->
    <color name="bg_epg_cell_focus">#2D3748</color>
</resources>
```

### Translucent Glassmorphism Card Style
Apply a thin, white transparent stroke around layout cards to make them pop against active background video streams:

```xml
<!-- res/drawable/bg_premium_glass_card.xml -->
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="@color/bg_glass_overlay" />
    <corners android:radius="12dp" />
    <stroke android:width="1dp" android:color="#26FFFFFF" /> <!-- 15% Translucent white border -->
</shape>
```

---

## 2. D-PAD & TV REMOTE UX PRINCIPLES (TV FIRST METRIC)

An Android TV app must be 100% functional via a 5-button D-pad remote (Up, Down, Left, Right, Center). Mouse pointers or touch mechanics must be treated as non-existent.

### The 3 Laws of Remote Navigation

1. **The Focus Scale Factor (Hover Feedback)**:
   Any interactive button, list item, or category channel row MUST visually expand when focused to guide the user's eyes from across the living room.
   * *Implementation*: Smoothly scale focused items by **1.05x** over a **150ms** transition.
   ```kotlin
   view.setOnFocusChangeListener { v, hasFocus ->
       val scale = if (hasFocus) 1.05f else 1.0f
       v.animate()
           .scaleX(scale)
           .scaleY(scale)
           .setDuration(150)
           .start()
   }
   ```
2. **Dynamic Grid Centering**:
   When scrolling through a large vertical list of channels or horizontal EPG program listings, the active focused cell must **stay locked in the screen center**, with the content moving behind it (rather than the cursor scrolling off the screen edges).
   * *Implementation*: Use `VerticalGridView` and `HorizontalGridView` (from Leanback) configured to lock scroll offsets exactly to the viewport midpoint.
3. **Auto-Dismiss Overlay Timing Rules**:
   Overlays must auto-dismiss gracefully so that active video streaming remains the hero visual state:
   * **Info Panel / Channel Bar**: **3 Seconds** (triggered on channel change).
   * **Quick Actions Overlay**: **5 Seconds** (triggered on D-pad Center click).
   * **Detailed Category Menus**: **10 Seconds** of inactivity before dismissing.
   * *Safety*: Always cancel active dismiss timers immediately if a user presses a navigation D-pad button, resetting the countdown.

---

## 3. LAYOUT OVERLAY STRUCTURES

TiViMate coordinates overlays as overlapping relative layers rather than loading heavy Android fragments, which would cause video playback frame drops.

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│             Layer 0: Active Video Stream               │
│             (SurfaceView / ExoPlayer Core)             │
│                                                        │
├────────────────────────────────────────────────────────┤
│ Layer 1: Info Panel / Channel Bar (Transparent Card)    │
│  [Logo]   Ch. 120 - Premium Movies                     │
│  [━━━━━━━━━────] 65% Completed                         │
│  1080p | AAC 2.0 | 60 fps                              │
└────────────────────────────────────────────────────────┘
```

---

## 4. PREMIUM UI INTERFACE DESIGN MOCKUP

To assist in visualizing these glassmorphic styles, dark HSL color palettes, and structured overlays in action, here is a custom visual reference mockup generated from TiViMate's UI parameters:

![Premium IPTV Interface Mockup](file:///C:/Users/elimi/.gemini/antigravity-ide/brain/52616612-5d7b-4e9d-b186-25e420ab8d1d/premium_iptv_overlay_ui_1779716685417.png)

---

## 5. CORE ENGINEERING BLUEPRINT SUMMARY

For maximum app fluidity, bind these five backend optimizations behind your custom visual layouts:

1. **Zero Database Queries in EPG Scroll**:
   Load EPG programs into sorted memory arrays once on guide launch. When scrolling the guide, perform binary-search queries on-the-fly to draw viewport-intersecting programs.
2. **Direct Canvas EPG Rendering**:
   Do not use RecyclerViews for EPG grids. Draw cells directly as calculated pixels on a hardware-accelerated Custom View Canvas.
3. **HTTP/2 Connection Pre-Warming**:
   Utilize the Media3 OkHttp datasource extension to keep socket connections pooled, reducing channel zapping times under 1 second.
4. **Streaming In-Memory Gzip Reader**:
   Stream, decompress, and parse EPG XMLTV data on-the-fly using `GZIPInputStream` directly to the XML tokenizer without writing temporary uncompressed files to disk.
5. **Foreground PVR Stream Dump**:
   Manage scheduled recordings via a Foreground Service holding a CPU `PARTIAL_WAKE_LOCK`, dumping HTTP Transport Stream packets directly to file buffers.
