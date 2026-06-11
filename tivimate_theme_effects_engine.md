# TiViMate Theme Engine & Visual Effects — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate's dynamic theme engine** and the suite of **premium visual effects** it renders. It explains how the app swaps visual color themes dynamically without interrupting active video playback, and dissects the specific graphic effects (focus glows, shadow elevations, glassmorphism, and fade-outs) that create its premium smart TV aesthetic.

---

## 1. THE DYNAMIC THEME OVERLAY WRAPPER ENGINE

In standard Android apps, changing a global theme requires calling `activity.recreate()`. On a smart TV app, recreating the activity destroys the active `ExoPlayer` instance, causing an annoying **3-to-5 second playback reload gap**.

TiViMate avoids this completely by running a **Dynamic Theme Overlay Wrapper** (`p363.WindowCallbackC4401` & `p363.C4425`).

```
[ Active MainActivity ]
          │
          ▼ (Intercepts Theme Change Event)
[ ContextThemeWrapper / WindowCallback (C4401) ]
          │
          ├─► Resolve Theme Attribute (R.attr.u_res_0x7f04000b)
          ├─► Instantiate fresh ContextThemeWrapper with new Style
          └─► Apply theme overlay to Window Callback directly
          
[ Active View Hierarchy ] ◄─── Broadcasts ─── [ Dynamic Drawable Update ]
(No ExoPlayer Re-instantiation = Zero Stream Stutter)
```

### The Theme-Swapping Handshake (`p363.WindowCallbackC4401.java`)

Instead of recreating the parent shell, TiViMate dynamically resolves the target overlay style resource from theme attributes and applies it directly to the active window wrapping context:

```java
// Decompiled theme resolving in WindowCallbackC4401.java (Line 387)
Resources.Theme theme = context.getTheme();
theme.resolveAttribute(R.attr.u_res_0x7f04000b, typedValue, true);

if (typedValue.resourceId != 0) {
    Resources.Theme themeNewTheme = context.getResources().newTheme();
    themeNewTheme.setTo(theme);
    themeNewTheme.applyStyle(typedValue.resourceId, true); // Override styling overlays on-the-fly
    c2219.getTheme().setTo(themeNewTheme);
}
```

This updates the active theme attributes (like focus colors, accents, and card stroke drawables). The views then refresh their drawables dynamically via `.invalidate()` or custom color-state-lists, while the underlying ExoPlayer media thread continues decoding packets completely undisturbed.

---

## 2. THE VISUAL EFFECTS SUITE (Living Room Aesthetics)

To deliver a premium interface, TiViMate combines micro-animations with complex overlapping alpha shapes that look outstanding on large 4K TV displays.

### Effect 1: Translucent Glassmorphism & Masking
*   **Aesthetic Goal**: Overlay panels (Channel Bar, Info Bar, Settings Panel) should feel connected to the video streaming behind them, rather than isolating the user behind a solid dark box.
*   **Decompiled Formula**:
    Using HSL tailored color overlays (Deep dark slate `#0B0F19` set to 80% opacity: `#CC0B0F19`), bordered by a subtle 1dp solid white stroke set to 15% opacity (`#26FFFFFF`).
*   **Replication Blueprint**:
    ```xml
    <!-- res/drawable/glass_panel_stroke.xml -->
    <shape xmlns:android="http://schemas.android.com/apk/res/android"
        android:shape="rectangle">
        <solid android:color="#CC0B0F19" /> <!-- 80% Slate Dark -->
        <corners android:radius="12dp" />
        <stroke android:width="1dp" android:color="#26FFFFFF" /> <!-- Thin semi-transparent boundary -->
    </shape>
    ```

### Effect 2: Dynamic D-pad Outline Glow (Active Focus)
*   **Aesthetic Goal**: High-contrast outline loops that glow around focused elements.
*   **Decompiled Formula**:
    Rather than displaying flat highlights, TiViMate uses **ColorStateList state triggers** that swap background layers. When focused, the card strokes transition from a muted transparent gray border directly to a glowing active blue accent (`#3B82F6`) or warm orange/teal (depending on the active theme resolved by `ContextThemeWrapper`).
*   **Replication Blueprint**:
    ```xml
    <!-- res/color/stroke_focus_state.xml -->
    <selector xmlns:android="http://schemas.android.com/apk/res/android">
        <item android:state_focused="true" android:color="#3B82F6" /> <!-- Glowing Accent -->
        <item android:state_focused="false" android:color="#26FFFFFF" /> <!-- Muted Border -->
    </selector>
    ```

### Effect 3: Micro-scale Elevation Elevations
*   **Aesthetic Goal**: Give focused items a 3D "lifted" depth effect.
*   **Decompiled Formula**:
    Upon receiving D-pad focus, the element's layout scale shifts smoothly. Concurrently, a subtle card elevation (shadow casting) is added to elevate the item above its neighbors.
*   **Replication Blueprint**:
    ```kotlin
    view.setOnFocusChangeListener { v, hasFocus ->
        val targetScale = if (hasFocus) 1.05f else 1.0f
        val targetElevation = if (hasFocus) 8f.dpToPx() else 2f.dpToPx()
        
        v.animate()
            .scaleX(targetScale)
            .scaleY(targetScale)
            .translationZ(targetElevation) // Elevates shadow cast layer dynamically
            .setDuration(150)
            .setInterpolator(DecelerateInterpolator())
            .start()
    }
    ```

### Effect 4: Smooth Alpha Cross-Fades
*   **Aesthetic Goal**: Avoid layout pops or jarring menu flashes.
*   **Decompiled Formula**:
    All UI panels fade in/out using a decelerated **alpha animation** curve locked exactly between **150ms and 250ms**. If a user presses a D-pad remote button during a fade, the animation is instantly canceled, resetting the opacity back to full immediately to ensure the remote input feels hyper-responsive.

---

## 3. HOW YOU CAN APPLY THESE PARADIGMS

### 🟢 What You CAN Use & Adapt:
*   **`ContextThemeWrapper` overlays**:
    Apply styles dynamically to views when the user changes a preference accent (e.g. from Blue to Teal), avoiding any global activity recreation.
*   **Color State List Drawables**:
    Use XML selector drawable parameters to manage focused border lines and text colors automatically at the Android system level.
*   **TranslationZ Elevations**:
    Utilize standard `translationZ` layout animators in Kotlin to elevate focused cards cleanly.

### 🔴 What You CANNOT Copy:
*   **Obfuscated Window Wrapper Classes**:
    Do not copy compiler-flattened wrappers like `WindowCallbackC4401.java`. They contain obfuscated interfaces generated to capture internal hardware key events and will fail to resolve. Use clean standard `ContextThemeWrapper` instances when inflating custom overlay views.
