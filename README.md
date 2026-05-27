# ✦ Dynamic Island for Android

A premium, iPhone-inspired Dynamic Island overlay for Android built with Jetpack Compose.
Smooth spring-physics animations, real system event listeners, and a clean modular architecture.

---

## Architecture Overview

```
Event Sources          Core Layer                    UI Layer
──────────────         ────────────────────────      ──────────────────────
CallStateReceiver  ─→  LiveActivityController  ─→   DynamicIslandComposable
MediaSessionConn   ─→  ActivityPriorityManager ─→     ├─ IdleIsland (empty pill)
BluetoothReceiver  ─→  DynamicIslandStateManager      ├─ CompactIsland
BatteryReceiver    ─→                           ─→     ├─ ExpandedIsland
NotificationList   ─→  (picks highest priority)        └─ TemporaryAlertIsland
TimerManager       ─→  (manages transitions)
                                                      Hosted in:
                        Overlay Service              DynamicIslandOverlayService
                        (foreground service,         (WindowManager TYPE_APPLICATION_OVERLAY)
                         always-on-top window)
```

---

## Setup

### 1. Permissions
Grant these at runtime before starting the service:

| Permission | How to Grant |
|---|---|
| `SYSTEM_ALERT_WINDOW` | Settings → Apps → Special Access → Display over other apps |
| Notification Access | Settings → Apps → Special App Access → Notification access |
| `READ_PHONE_STATE` | Standard runtime permission dialog |
| `BLUETOOTH_CONNECT` | Standard runtime permission dialog (Android 12+) |

### 2. Build & Install
```bash
./gradlew assembleDebug
adb install app/build/outputs/apk/debug/app-debug.apk
```

### 3. First Launch
1. Open the app
2. Tap **Grant** on the overlay permission card
3. Go to **Settings → Notification Access** and enable **Dynamic Island**
4. Toggle the **Island** switch ON
5. The island will appear near your front camera

---

## Customising Island Size & Position

All size constants live in `IslandDisplayConfig`:

```kotlin
// In DynamicIslandOverlayService.kt — change islandConfig:
private val islandConfig = IslandDisplayConfig(
    topOffsetDp = 14f,          // ← move up/down (px from top)
    idleWidthDp = 120f,         // ← idle pill width
    idleHeightDp = 34f,         // ← idle pill height
    compactWidthDp = 240f,      // ← compact state width
    compactHeightDp = 36f,      // ← compact state height
    expandedWidthFraction = 0.88f, // ← 88% of screen width
    expandedHeightDp = 120f,    // ← expanded height
    cornerRadiusDp = 20f        // ← pill corner radius
)
```

**Device-specific presets:**
```kotlin
IslandDisplayConfig.Default           // General — works on most phones
IslandDisplayConfig.CenteredPunchHole // Vivo X200 Pro, Pixel 8, etc.
IslandDisplayConfig.Teardrop          // Teardrop notch phones
```

---

## Enabling / Disabling Activity Modules

Each activity type is independent. Disable any you don't need:

```kotlin
// In your SettingsRepository or via the Settings screen:
settingsRepository.setModuleEnabled("media", false)      // disable media
settingsRepository.setModuleEnabled("charging", false)   // disable charging alert
settingsRepository.setModuleEnabled("hotspot", false)    // disable hotspot

// Available modules:
// "calls" | "media" | "navigation" | "timer"
// "charging" | "earbuds" | "hotspot" | "biometric" | "alerts"
```

---

## Pushing Custom Events

Use `LiveActivityController` from anywhere in your app:

```kotlin
@Inject lateinit var controller: LiveActivityController

// Show a quick alert
controller.showAlert("Payment successful!", AlertType.SUCCESS)

// Start a timer (use TimerManager for auto-ticking)
timerManager.startTimer("Pomodoro", durationSeconds = 1500L)

// Update navigation
controller.onNavigationUpdate(
    nextInstruction = "Turn right on Oak Ave",
    distanceToNext = "0.8 mi",
    eta = "6 min",
    turnDirection = TurnDirection.RIGHT
)
```

---

## Island States

| State | Trigger | Behaviour |
|---|---|---|
| `Idle` | No active event | Minimal empty pill. **No fake content.** |
| `Compact` | Event starts | Small pill with live info row |
| `Expanded` | User taps island | Full panel with rich detail |
| `TemporaryAlert` | Auto-dismiss events | Brief compact view, auto-collapses |

**Activity Priority** (highest to lowest):
1. Phone / Video Call
2. Navigation
3. Timer
4. Media Playback *(only when actually playing)*
5. File Transfer
6. System (Charging, Earbuds, Hotspot, Biometric)
7. Quick Alerts

---

## File Structure

```
app/src/main/java/com/dynamicisland/android/
├── DynamicIslandApp.kt              Hilt Application
├── DynamicIslandModule.kt           Hilt DI module
├── MainActivity.kt                  Setup UI + demo controls
│
├── data/
│   ├── LiveActivity.kt              All activity data models
│   └── IslandState.kt               UI state + display config
│
├── controller/
│   └── LiveActivityController.kt    Public API for pushing events
│
├── manager/
│   ├── ActivityPriorityManager.kt   Multi-activity priority arbiter
│   ├── DynamicIslandStateManager.kt State machine + transitions
│   └── TimerManager.kt              Countdown timer logic
│
├── service/
│   └── DynamicIslandOverlayService.kt  Foreground service + overlay window
│
├── listeners/
│   ├── CallStateListener.kt         Phone call events
│   ├── MediaSessionConnector.kt     Real media playback detection
│   ├── NotificationEventListener.kt NotificationListenerService
│   └── SystemListeners.kt           Battery, Bluetooth, Boot
│
├── settings/
│   └── SettingsRepository.kt        DataStore preferences
│
└── ui/
    ├── animation/
    │   └── AnimationController.kt   Spring specs + transitions
    ├── components/
    │   ├── IslandBackground.kt       Animated pill container
    │   ├── CompactIsland.kt          Compact content per activity
    │   ├── ExpandedIsland.kt         Expanded panels per activity
    │   └── DynamicIslandComposable.kt Root composable
    └── theme/
        └── IslandTheme.kt            Colors, typography, spacing
```

---

## Key Design Decisions

### No fake content in idle state
`IdleIslandContent` is intentionally empty. The black pill is the idle state.
No music timer, no waveform, no fake player when nothing is happening.

### Media only shown when actually playing
`MediaSessionConnector` explicitly calls `controller.onMediaPaused()` for
`STATE_PAUSED` and `STATE_STOPPED`. The island never shows a media card unless
`PlaybackState.STATE_PLAYING` is active.

### Spring-physics animations
All size/shape transitions use `spring()` with tuned `dampingRatio` and `stiffness`
values in `IslandAnimationSpecs`. This produces the characteristic iOS Dynamic Island
"living" feel — no linear tweens for shape changes.

### Overlay lifecycle
`OverlayLifecycleOwner` provides a synthetic `LifecycleOwner` for the `ComposeView`
running inside the `WindowManager` overlay, outside of any Activity.

---

## Tested On
- Vivo X200 Pro (primary target — centred punch-hole)
- Pixel 7 / 8 (centred punch-hole)
- Samsung Galaxy S23 (top-center punch-hole)
- Android 10 → 14

---

## Permissions Summary

```xml
SYSTEM_ALERT_WINDOW         <!-- Draw overlay on top of apps -->
FOREGROUND_SERVICE          <!-- Keep service alive -->
READ_PHONE_STATE            <!-- Detect calls -->
BIND_NOTIFICATION_LISTENER_SERVICE <!-- Media sessions + alerts -->
BLUETOOTH_CONNECT           <!-- Earbuds detection -->
ACCESS_FINE_LOCATION        <!-- Navigation (optional) -->
RECEIVE_BOOT_COMPLETED      <!-- Auto-restart after reboot -->
VIBRATE                     <!-- Call/biometric feedback -->
ACCESS_WIFI_STATE           <!-- Hotspot status -->
```
