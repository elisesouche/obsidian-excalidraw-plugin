# Boox Stylus API Research: boox-rapid-draw Analysis

> **Source repository:** [sergeylappo/boox-rapid-draw](https://github.com/sergeylappo/boox-rapid-draw)  
> **Purpose of analysis:** Understand how the Onyx/Boox SDK is used so we can plan integrating Boox stylus support into the Obsidian Excalidraw plugin.

---

## 1. What is boox-rapid-draw?

`boox-rapid-draw` is an Android application that runs as a **transparent overlay foreground service** on Boox (Onyx International) e-ink devices. When active, it intercepts stylus input and routes it through the Boox/Onyx proprietary SDK so that strokes are drawn by the **e-ink hardware controller** instead of Android's normal compositing pipeline. The result is near-zero-latency ink rendering on any app, because the E-ink display writes pixels directly for the stylus path without waiting for Android to composite a frame.

The project uses two Onyx SDK Maven libraries:

| Artifact | Version (used in project) |
|---|---|
| `com.onyx.android.sdk:onyxsdk-device` | 1.2.30 |
| `com.onyx.android.sdk:onyxsdk-pen` | 1.4.11 |

An additional library `org.lsposed.hiddenapibypass:hiddenapibypass` (v4.3) is needed to allow access to Android hidden APIs on Android 11+ (`Build.VERSION.SDK_INT >= R`). This is called once in `Application.onCreate()` via `HiddenApiBypass.addHiddenApiExemptions("")`.

---

## 2. Core API Surface

### 2.1 `TouchHelper` (package `com.onyx.android.sdk.pen`)

This is the central object. It bridges an Android `SurfaceView` to the Onyx e-ink pen subsystem.

**Creation**

```kotlin
val touchHelper = TouchHelper.create(surfaceView: SurfaceView, version: Int, callback: RawInputCallback)
```

- `surfaceView` – the view the SDK will draw on and capture stylus events from.
- `version` – API version integer (the project uses `2`).
- `callback` – a `RawInputCallback` instance that receives all stylus lifecycle events.

**Configuration (called inside the view's `onLayoutChange` listener)**

```kotlin
touchHelper.setStrokeColor(Color.BLACK)
touchHelper.setStrokeStyle(TouchHelper.STROKE_STYLE_PENCIL)
touchHelper.openRawDrawing()                        // activates hardware-accelerated drawing
touchHelper.setStrokeWidth(strokeWidth)
    .setLimitRect(bounds, exclusionList)            // defines drawable region
touchHelper.setRawInputReaderEnable(!touchHelper.isRawDrawingInputEnabled)
```

- `openRawDrawing()` – switches the Boox display controller into "raw drawing" mode. In this mode, the E-ink panel writes stylus strokes at its own hardware refresh rate (highest possible speed), bypassing the Android UI thread.
- `setLimitRect(bounds, exclusions)` – limits where the SDK accepts stylus input. Exclusion rectangles can be provided (e.g. toolbars, menus).
- `setRawInputReaderEnable(enable)` – enables/disables whether the raw input reader thread is active. The project toggles this on layout changes to reset the reader after device rotation.

**Pen-up refresh timing**

```kotlin
touchHelper.setPenUpRefreshTimeMs(1000)
```

Controls how many milliseconds after the stylus is lifted before the E-ink display performs a full (crisp) refresh. Lower values = faster refresh after stroke, but may cause visible ghosting. The project uses 1 second.

**In the `onPenActive` callback**

```kotlin
override fun onPenActive(point: TouchPoint?) {
    touchHelper.setRawDrawingEnabled(true)
}
```

`setRawDrawingEnabled(true)` must be called when the pen is detected to start actual stroke rendering. If not called, the SDK is listening but not drawing.

**Cleanup**

```kotlin
touchHelper.closeRawDrawing()   // called in Service.onDestroy()
```

Shuts down the pen subsystem and releases hardware resources.

**Key boolean properties**

| Property / Method | Meaning |
|---|---|
| `isRawDrawingInputEnabled` | Read-only: whether raw input reading is active |
| `isRawDrawingRenderEnabled = false` | Write: temporarily disables raw rendering (used during `onPenUpRefresh` to prevent flicker) |

---

### 2.2 `RawInputCallback` (package `com.onyx.android.sdk.pen`)

Abstract base class. Subclass and override the relevant methods. All methods receive either a `TouchPoint` or `TouchPointList`.

```kotlin
val callback = object : RawInputCallback() {

    // Pen stroke lifecycle
    override fun onBeginRawDrawing(success: Boolean, touchPoint: TouchPoint?) {}
    override fun onEndRawDrawing(success: Boolean, touchPoint: TouchPoint?) {}
    override fun onRawDrawingTouchPointMoveReceived(touchPoint: TouchPoint?) {}
    override fun onRawDrawingTouchPointListReceived(touchPointList: TouchPointList) {}

    // Pen detection
    override fun onPenActive(point: TouchPoint?) {
        // Pen hover detected – enable drawing
        touchHelper.setRawDrawingEnabled(true)
    }

    // Eraser lifecycle (barrel button or rubber end)
    override fun onBeginRawErasing(success: Boolean, touchPoint: TouchPoint?) {}
    override fun onEndRawErasing(success: Boolean, touchPoint: TouchPoint?) {}
    override fun onRawErasingTouchPointMoveReceived(touchPoint: TouchPoint?) {}
    override fun onRawErasingTouchPointListReceived(touchPointList: TouchPointList?) {}

    // Display refresh
    override fun onPenUpRefresh(refreshRect: RectF?) {
        // Disable rendering during E-ink refresh to prevent flicker
        touchHelper.isRawDrawingRenderEnabled = false
        super.onPenUpRefresh(refreshRect)
    }
}
```

**Important:** In `boox-rapid-draw`, `onRawDrawingTouchPointMoveReceived` and `onRawDrawingTouchPointListReceived` are **left empty** intentionally. This is because in this app the SDK itself draws the strokes via the hardware; there is no need to process them in software. For integration with Excalidraw, **these are the key callbacks** – we would intercept the points here and forward them to the Excalidraw canvas.

---

### 2.3 `TouchPoint` (package `com.onyx.android.sdk.data.note`)

Represents a single stylus sample. Expected fields (based on Onyx SDK documentation and usage context):

| Field | Type | Description |
|---|---|---|
| `x` | `float` | Screen X coordinate (pixels) |
| `y` | `float` | Screen Y coordinate (pixels) |
| `pressure` | `float` | Tip pressure (0.0–1.0) |
| `size` | `float` | Contact area |
| `timestamp` | `long` | Event timestamp (ms) |
| `tiltX` / `tiltY` | `float` | Pen tilt angles (may vary by device) |

> **Note:** The exact field names depend on the SDK version. The `onyxsdk-pen` library is not public on Maven Central; it is distributed by Onyx on their developer portal. The fields above match the general Onyx pen API contract.

---

### 2.4 `TouchPointList` (package `com.onyx.android.sdk.pen.data`)

A batch container of `TouchPoint` objects collected between `onBeginRawDrawing` and `onEndRawDrawing`. Accessing individual points is typically done via its iterator or a `getPoints()` method.

---

### 2.5 `RxManager` (package `com.onyx.android.sdk.rx`)

Called once in `Application.onCreate()`:

```kotlin
RxManager.Builder.initAppContext(this)
```

Required initialization for the Onyx SDK's internal RxJava-based event bus. Without this, `TouchHelper` will not function.

---

## 3. Application Architecture

```
BooxRapidDraw (Application)
  └─ onCreate()
       ├─ RxManager.Builder.initAppContext(this)
       └─ HiddenApiBypass.addHiddenApiExemptions("") [Android 11+]

MainActivity (FragmentActivity)
  └─ if overlay permission granted → start/stop OverlayShowingService
  └─ if not → show permission dialog → redirect to Settings

OverlayShowingService (Foreground Service)
  ├─ Foreground notification (with "Stop" action)
  ├─ SurfaceView overlay (TYPE_APPLICATION_OVERLAY, transparent, FLAG_NOT_TOUCHABLE)
  └─ TouchHelper
       └─ RawInputCallback
            ├─ onPenActive → setRawDrawingEnabled(true)
            └─ onPenUpRefresh → isRawDrawingRenderEnabled = false

RapidDrawTileService (TileService)
  └─ Quick Settings tile → toggle OverlayShowingService via MainActivity intent
```

**Key design choices:**

1. **Overlay is `FLAG_NOT_TOUCHABLE`** – touch events pass through to the underlying app. The SurfaceView captures stylus data only through the Onyx SDK (which operates at driver level, below Android's touch dispatch).

2. **SurfaceView uses `PixelFormat.TRANSPARENT`** – the overlay canvas is visually transparent so the user sees the underlying app; only SDK-drawn ink strokes appear.

3. **`setZOrderOnTop(true)`** – ensures the SurfaceView's surface is rendered above all other window content.

4. **Service restarts on kill** – `START_STICKY` makes Android restart the service if killed (e.g. low memory). `START_NOT_STICKY` is returned when the STOP action is received to prevent restart.

---

## 4. AndroidManifest Requirements

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

<service
    android:name=".OverlayShowingService"
    android:foregroundServiceType="specialUse">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="This app needs to display itself over other APPs to emulate quick draw." />
</service>
```

The `SYSTEM_ALERT_WINDOW` permission (overlay permission) must be granted by the user via Settings. This is the same permission that apps like screen recorders and floating windows use.

---

## 5. The E-ink Raw Drawing Mechanism Explained

On a normal Android device, drawing goes through:

```
App → Android Canvas / OpenGL → SurfaceFlinger → Display
```

On Boox E-ink devices, the Onyx SDK short-circuits this to:

```
Stylus hardware → Onyx pen driver → E-ink controller → Pixels directly
```

This is why **there is no lag** – the E-ink panel's own refresh engine draws the ink as fast as the hardware allows (typically 40–120 fps in "fast" mode for a small area), while the Android UI thread processes events at 60 fps (or less).

The `openRawDrawing()` call is what enables this hardware path. When active:
- The SDK intercepts pen events at driver level before Android's `MotionEvent` system sees them.
- Pixels are written to the display by the E-ink controller.
- The `SurfaceView` canvas is used only as a coordinate reference.

When the pen is lifted (`onPenUpRefresh`), the display controller does a "waveform refresh" to clear ghosting. This is why `isRawDrawingRenderEnabled = false` is set – to prevent a software-drawn duplicate of the stroke from appearing during that brief hardware refresh window.

---

## 6. What the SDK Does NOT Do (Relevant for Integration)

- It does **not** transmit stylus data over any network protocol.
- It does **not** expose a JavaScript or web API.
- It does **not** work outside Android on Boox/Onyx hardware.
- The raw drawing goes directly to the display hardware; it does **not** persist as vector data unless the application code explicitly processes `TouchPointList` in the callback.

---

## 7. Key Takeaway for Excalidraw Integration

The boox-rapid-draw project shows the complete minimal pattern to:

1. Receive stylus `TouchPoint` data (x, y, pressure, tilt) from the Onyx SDK.
2. Render strokes with zero lag on the E-ink display.

For Excalidraw integration, we need to capture the `TouchPointList` data from `onRawDrawingTouchPointListReceived` and convert it into synthetic `PointerEvent`s (or use a WebView bridge) that Excalidraw can consume. The integration plan is in [`boox-stylus-integration-plan.md`](./boox-stylus-integration-plan.md).
