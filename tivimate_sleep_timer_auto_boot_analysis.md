# TiViMate Sleep Timer & Auto-Boot Lifecycle

This document outlines the architecture behind TiViMate's automated lifecycle features, specifically how it manages the sleep timer, automatically starting on device boot, and waking the device from sleep.

---

## 1. THE SLEEP TIMER

TiViMate includes a native sleep timer to automatically stop playback and exit the app (or put the device to sleep) after a set duration.

### 1.1 UI and Triggers

**Key Strings:**
* `u_res_0x7f1301bf` = **"Sleep timer"**
* `u_res_0x7f1301c2` = **"Sleep timer is off"**
* `u_res_0x7f1301c3` = **"Sleep timer is on"**

The sleep timer is accessed via the Quick Actions OSD (the control panel shown over active video playback). Users can select predefined countdowns (e.g., 15m, 30m, 60m, 120m).

### 1.2 Underlying Mechanism

When a timer is set, TiViMate utilizes the Android `Handler` or `AlarmManager` mechanism.

1. **Countdown State:** The app stores the target stop time (current epoch + duration) in its internal state.
2. **Tick Updates:** If the user opens the Quick Actions menu while a timer is active, the icon/text dynamically updates to show the remaining time.
3. **Execution:** When the timer expires, the app triggers a shutdown sequence:
   * It calls `player.stop()` on the ExoPlayer instance.
   * It executes `Activity.finishAndRemoveTask()` to cleanly exit the app.
   * *Optional (Device dependent):* If the app has Device Administrator privileges or relies on HDMI-CEC, it may attempt to turn off the TV/box screen.

---

## 2. AUTO-BOOT & WAKE LIFECYCLE

For dedicated IPTV boxes, users often want the device to boot straight into live TV, bypassing the Android TV launcher entirely. TiViMate supports this via the `RECEIVE_BOOT_COMPLETED` permission.

### 2.1 Auto-Start on Boot

**Key String:**
* `u_res_0x7f13026c` = **"Auto start app on boot"**

**Implementation:**
TiViMate registers a `BroadcastReceiver` in its `AndroidManifest.xml` listening for the `android.intent.action.BOOT_COMPLETED` intent.

When the Android OS finishes booting, it broadcasts this intent. If the user has enabled the "Auto start" setting, the receiver intercepts the broadcast and launches `MainActivity`.

*Note:* On modern Android versions (Android 10+), launching activities directly from the background is heavily restricted. TiViMate likely uses the `SYSTEM_ALERT_WINDOW` permission or relies on Android TV-specific allowances to ensure the app foregrounds successfully.

### 2.2 Wake from Sleep

**Key Strings:**
* `u_res_0x7f130217` = **"Wake up from sleep mode"**
* `u_res_0x7f13026d` = **"Auto start app on wake up from sleep mode"**

Many Android TV boxes don't actually turn off; they enter a low-power "sleep" or "suspend" mode.

**Implementation:**
To detect waking from sleep, TiViMate listens for the `android.intent.action.SCREEN_ON` or `android.intent.action.USER_PRESENT` broadcasts.

When the screen turns on (e.g., the user presses the power button on their remote), the app intercepts the intent. If the setting is enabled, it forces `MainActivity` to the foreground, resuming playback of the last active channel.

### 2.3 Autoplay on Launch

**Key String:**
* `u_res_0x7f130270` = **"Autoplay channels"**

Coupled with the auto-boot features, TiViMate remembers the `last_turn_on_time` for channels (see `C3210.java` schema). Upon startup, if "Autoplay channels" is enabled, the app automatically tunes to the last watched channel instead of waiting on the TV Guide screen.
