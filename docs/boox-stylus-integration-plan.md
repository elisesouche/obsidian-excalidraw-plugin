# Integration Plan: Boox Stylus API → Obsidian Excalidraw Plugin

> **Related research:** [`boox-stylus-api-research.md`](./boox-stylus-api-research.md)  
> **Goal:** Make the Obsidian Excalidraw plugin fully responsive to Boox/Onyx stylus input on Android, delivering low-latency, pressure-sensitive freehand drawing.

---

## Executive Summary

Boox devices run Android. Obsidian on Android runs as a standard Android app whose UI is rendered in a WebView. The Excalidraw plugin renders inside that WebView. The Onyx SDK (`onyxsdk-pen`) is a native Android library that must run in Kotlin/Java code inside the host Android process.

The integration therefore has two distinct layers:

1. **Native Android layer** – Kotlin code that uses `TouchHelper` / `RawInputCallback` to capture stylus events from the Onyx SDK.
2. **WebView / JavaScript layer** – TypeScript code that receives those events and injects them into the Excalidraw React component as synthetic `PointerEvent`s.

These two layers must communicate via a bridge (either a `JavascriptInterface`, a WebSocket, or a WebView message channel).

Because Obsidian is a closed-source app whose Android wrapper cannot be modified directly, the integration must work within the plugin system. The plan below covers two implementation tracks, ordered from most realistic to least:

- **Track A (Preferred):** A companion Android app/service running alongside Obsidian that bridges stylus events via a local WebSocket.
- **Track B (Future):** A native Obsidian Android plugin extension if/when Obsidian exposes such a mechanism.

---

## Context: How Excalidraw Receives Input Today

Excalidraw (the React component wrapped by this plugin) uses standard browser Pointer Events API:

- `PointerEvent.pointerType` (`"pen"` | `"touch"` | `"mouse"`)
- `PointerEvent.pressure` (0.0–1.0)
- `PointerEvent.tiltX`, `PointerEvent.tiltY`
- `PointerEvent.pointerId`

When `pointerType === "pen"`, Excalidraw uses the pressure value to modulate stroke width (via the `perfect-freehand` library, configured through `PenOptions` / `StrokeOptions`). The plugin already has a rich pen settings system (`PenSettingsModal`, `PenStyle`, `PENS`, `dynamicStyling`).

The existing pen mode settings in the plugin (`defaultPenMode`, `penModeDoubleTapEraser`, `penModeSingleFingerPanning`) already handle the concept of a physical stylus but rely on the WebView receiving genuine `PointerEvent`s with `pointerType === "pen"`.

The core problem on Boox is that the Onyx SDK intercepts stylus events at the **driver level** before Android's input system dispatches them as normal `MotionEvent`s. The WebView therefore never receives `pointerType === "pen"` events from the Boox stylus – it gets nothing, or at best touch events.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Boox Device (Android)                               │
│                                                      │
│  ┌─────────────────────┐    local WebSocket          │
│  │  Boox Bridge App    │◄──────────────────────────┐ │
│  │  (Kotlin/Android)   │                            │ │
│  │                     │   TouchPoint events        │ │
│  │  TouchHelper        │──────────────────────────►│ │
│  │  RawInputCallback   │                            │ │
│  └─────────────────────┘                            │ │
│                                                     │ │
│  ┌───────────────────────────────────────────────┐  │ │
│  │  Obsidian App (Android WebView)               │  │ │
│  │                                               │  │ │
│  │  ┌─────────────────────────────────────────┐ │  │ │
│  │  │  Excalidraw Plugin (TypeScript)         │ │  │ │
│  │  │                                         │ │  │ │
│  │  │  BooxStylusReceiver (new module)        │◄┘  │ │
│  │  │    ↓ synthetic PointerEvents            │ │  │ │
│  │  │  Excalidraw React component             │ │  │ │
│  │  │    ↓ onPointerUpdate callback           │ │  │ │
│  │  │  ExcalidrawView.ts                      │ │  │ │
│  │  └─────────────────────────────────────────┘ │  │ │
│  └───────────────────────────────────────────────┘  │ │
└─────────────────────────────────────────────────────┘
```

---

## Track A: Companion Android App (WebSocket Bridge)

This track creates a small standalone Android app (similar to `boox-rapid-draw`) that:
1. Captures stylus events via the Onyx SDK.
2. Forwards them to a local WebSocket server that the Excalidraw plugin connects to.
3. Does **not** use the hardware drawing path (`openRawDrawing`) – that would conflict with Excalidraw rendering.

### Phase 1: Companion Android App

#### 1.1 Project Setup

- Create a new Android project (Kotlin, `minSdk 28`, `targetSdk 34`).
- Add Gradle dependencies:
  ```toml
  onyx-pen = { group = "com.onyx.android.sdk", name = "onyxsdk-pen", version = "1.4.11" }
  onyx-device = { group = "com.onyx.android.sdk", name = "onyxsdk-device", version = "1.2.30" }
  hiddenapibypass = { module = "org.lsposed.hiddenapibypass:hiddenapibypass", version = "4.3" }
  ktor-server-websockets = { ... }   # or java-websocket / OkHttp
  ```
- Initialize `RxManager` and `HiddenApiBypass` in `Application.onCreate()` (same as `boox-rapid-draw`).
- Declare `SYSTEM_ALERT_WINDOW` permission in `AndroidManifest.xml`.

#### 1.2 Overlay SurfaceView Service

Create a `StylusBridgeService` (Foreground Service, identical pattern to `OverlayShowingService`):

```kotlin
class StylusBridgeService : Service() {
    private lateinit var touchHelper: TouchHelper
    private lateinit var overlayView: SurfaceView
    private lateinit var wsServer: StylusWebSocketServer  // see 1.3

    override fun onCreate() {
        super.onCreate()
        startForegroundNotification()
        createOverlay()         // transparent, FLAG_NOT_TOUCHABLE overlay
        initTouchHelper()
        wsServer.start()
    }

    private fun initTouchHelper() {
        touchHelper = TouchHelper.create(overlayView, 2, object : RawInputCallback() {

            override fun onPenActive(point: TouchPoint?) {
                touchHelper.setRawDrawingEnabled(true)
                // NOTE: do NOT call openRawDrawing() here – let Excalidraw render
            }

            override fun onBeginRawDrawing(b: Boolean, p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "pointerdown", point = p))
            }

            override fun onRawDrawingTouchPointMoveReceived(p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "pointermove", point = p))
            }

            override fun onRawDrawingTouchPointListReceived(list: TouchPointList) {
                list.points.forEach { p ->
                    wsServer.send(StylusEvent(type = "pointermove", point = p))
                }
            }

            override fun onEndRawDrawing(b: Boolean, p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "pointerup", point = p))
                touchHelper.setRawDrawingEnabled(false)
            }

            override fun onPenUpRefresh(rect: RectF?) {
                // Do NOT disable rendering here – Excalidraw owns the display
                super.onPenUpRefresh(rect)
            }

            // Eraser events → translate to Excalidraw eraser tool
            override fun onBeginRawErasing(b: Boolean, p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "eraser_begin", point = p))
            }
            override fun onRawErasingTouchPointMoveReceived(p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "eraser_move", point = p))
            }
            override fun onEndRawErasing(b: Boolean, p: TouchPoint?) {
                wsServer.send(StylusEvent(type = "eraser_end", point = p))
            }
        })

        overlayView.addOnLayoutChangeListener { v, left, top, right, bottom, _, _, _, _ ->
            val bounds = Rect(0, 0, right, bottom)
            touchHelper.setStrokeColor(Color.TRANSPARENT)  // invisible – Excalidraw draws
            touchHelper.openRawDrawing()
            touchHelper.setStrokeWidth(0.1f).setLimitRect(bounds, listOf())
            touchHelper.setRawInputReaderEnable(true)
        }
    }
}
```

**Critical difference from boox-rapid-draw:** We still call `openRawDrawing()` to activate the driver-level event capture, but we set `strokeColor` to transparent and keep stroke width minimal. The SDK must be in raw drawing mode to intercept stylus events before Android sees them; we just do not let it render anything visible so Excalidraw can render its own strokes.

> ⚠️ **Risk:** When `openRawDrawing()` is active, the Onyx display controller may suppress normal Android touch dispatch. This needs careful testing. An alternative is to use only the input-reader path without full raw drawing mode, which may reduce performance but avoid rendering conflicts.

#### 1.3 WebSocket Server

A minimal local WebSocket server (port 7854, chosen to avoid conflicts):

```kotlin
data class StylusEvent(
    val type: String,          // "pointerdown" | "pointermove" | "pointerup" | "eraser_*"
    val x: Float,              // screen-absolute physical pixels (from TouchPoint)
    val y: Float,              // screen-absolute physical pixels (from TouchPoint)
    val pressure: Float,       // 0.0–1.0
    val tiltX: Float,
    val tiltY: Float,
    val timestamp: Long,
    // Coordinate translation fields (W1 fix) — computed once per session/orientation change:
    val webViewOffsetX: Float, // left edge of the WebView in physical screen pixels
    val webViewOffsetY: Float, // top edge of the WebView in physical screen pixels (≈ status bar height)
    val dpr: Float,            // display density (= window.devicePixelRatio in the WebView)
)
```

The companion app computes `webViewOffsetX/Y` and `dpr` once on startup and on every orientation change:

```kotlin
private fun computeWebViewGeometry(): Triple<Float, Float, Float> {
    val metrics = resources.displayMetrics
    val dpr = metrics.density                     // matches window.devicePixelRatio in WebView
    val rect = Rect()
    overlayView.getWindowVisibleDisplayFrame(rect) // rect.top = status bar height
    return Triple(rect.left.toFloat(), rect.top.toFloat(), dpr)
}
```

JSON wire format example:
```json
{
  "type": "pointermove",
  "x": 825.0,
  "y": 1606.4,
  "pressure": 0.65,
  "tiltX": 12.0,
  "tiltY": -5.0,
  "timestamp": 1714850000123,
  "webViewOffsetX": 0.0,
  "webViewOffsetY": 72.0,
  "dpr": 2.0
}
```

`x` and `y` are raw screen-absolute physical pixels from the Onyx SDK. The TypeScript receiver converts them to CSS viewport coordinates using `webViewOffsetX/Y` and `dpr` (see §2.1).

Use `java-websocket` library (`org.java-websocket:Java-WebSocket`) or Ktor for the server. Bind only to `localhost` (127.0.0.1) for security.

#### 1.4 User Interface

Simple toggle (same pattern as `boox-rapid-draw`):
- App icon click → start/stop the service.
- Quick Settings tile via `TileService`.
- Foreground notification with "Stop" button.

---

### Phase 2: Excalidraw Plugin – TypeScript WebSocket Receiver

A new module in the plugin that connects to the companion app's WebSocket and translates messages into synthetic `PointerEvent`s on the Excalidraw canvas element.

#### 2.1 New file: `src/utils/booxStylusReceiver.ts`

```typescript
// Responsible for:
// 1. Connecting to the companion app WebSocket
// 2. Translating StylusEvent messages into synthetic PointerEvents
// 3. Dispatching those events onto the canvas element

interface StylusEventMessage {
  type: string;
  x: number;              // screen-absolute physical pixels
  y: number;              // screen-absolute physical pixels
  pressure: number;
  tiltX: number;
  tiltY: number;
  timestamp: number;
  webViewOffsetX: number; // physical pixels from screen left to WebView left edge
  webViewOffsetY: number; // physical pixels from screen top to WebView top edge (≈ status bar)
  dpr: number;            // display density = window.devicePixelRatio
}

const PEN_POINTER_ID = 1;
const ERASER_POINTER_ID = 2;

export class BooxStylusReceiver {
  private ws: WebSocket | null = null;
  private canvas: HTMLCanvasElement | null = null;
  private enabled: boolean = false;       // W2 fix: tracks intent to stay connected
  private strokeInProgress: boolean = false; // W3 fix: tracks mid-stroke state

  connect(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.enabled = true;                  // W2 fix: set before opening the socket
    this.ws = new WebSocket("ws://127.0.0.1:7854");
    this.ws.onmessage = (evt) => this.onMessage(JSON.parse(evt.data));
    this.ws.onclose = () => {
      // W3 fix: cancel any in-progress stroke so Excalidraw doesn't get stuck
      if (this.strokeInProgress) {
        this.canvas?.dispatchEvent(new PointerEvent("pointercancel", {
          bubbles: true,
          pointerType: "pen",
          pointerId: PEN_POINTER_ID,
        }));
        this.strokeInProgress = false;
      }
      setTimeout(() => this.reconnect(), 2000);
    };
  }

  private reconnect() {
    if (this.enabled) this.connect(this.canvas!); // W2 fix: now actually executes
  }

  private onMessage(event: StylusEventMessage) {
    if (!this.canvas) return;

    // W1 fix: convert screen-absolute physical pixels → CSS viewport pixels.
    // webViewOffsetX/Y is the physical-pixel position of the WebView's top-left corner
    // on the physical screen (status bar height for Y, 0 for X on most devices).
    // dpr converts physical pixels to CSS pixels (same as window.devicePixelRatio).
    const clientX = (event.x - event.webViewOffsetX) / event.dpr;
    const clientY = (event.y - event.webViewOffsetY) / event.dpr;

    const isEraser = event.type.startsWith("eraser_");
    const pointerId = isEraser ? ERASER_POINTER_ID : PEN_POINTER_ID;

    // Map event type to PointerEvent type
    const pointerEventType = pointerEventTypeMap[event.type];
    if (!pointerEventType) return;

    // W3 fix: track stroke state so we can cancel on disconnect
    if (pointerEventType === "pointerdown") this.strokeInProgress = true;
    if (pointerEventType === "pointerup")   this.strokeInProgress = false;

    const syntheticEvent = new PointerEvent(pointerEventType, {
      bubbles: true,
      cancelable: true,
      pointerType: "pen",
      pointerId,          // W10 fix: distinct IDs for pen tip vs. eraser end
      clientX,
      clientY,
      pressure: event.pressure,
      tiltX: event.tiltX,
      tiltY: event.tiltY,
    });
    this.canvas.dispatchEvent(syntheticEvent);
  }

  disconnect() {
    this.enabled = false;               // W2 fix: prevent reconnect after explicit disconnect
    this.ws?.close();
    this.ws = null;
  }
}

const pointerEventTypeMap: Record<string, string> = {
  pointerdown:   "pointerdown",
  pointermove:   "pointermove",
  pointerup:     "pointerup",
  eraser_begin:  "pointerdown",   // switch tool to eraser first (see §2.4)
  eraser_move:   "pointermove",
  eraser_end:    "pointerup",
};
```

#### 2.2 Integration into `ExcalidrawView.ts`

In the `onExcalidrawInitialize` callback (where the `ExcalidrawImperativeAPI` becomes available):

```typescript
// After Excalidraw mounts and the canvas is available
const canvas = this.excalidrawContainer?.querySelector("canvas.interactive");
if (canvas && this.plugin.settings.booxStylusEnabled) {
  this.booxReceiver = new BooxStylusReceiver();
  this.booxReceiver.connect(canvas as HTMLCanvasElement);
}
```

In `onunload()` / `onClose()`:
```typescript
this.booxReceiver?.disconnect();
```

#### 2.3 Coordinate System Translation

The companion app's `TouchPoint` coordinates are **screen-absolute physical pixels**. A `PointerEvent.clientX/Y` must be in **CSS viewport pixels** (origin = top-left of the WebView viewport). The conversion requires two pieces of information, both provided by the companion app in the wire protocol (see §1.3):

1. **`webViewOffsetX/Y`** — the physical-pixel position of the WebView's top-left corner on the physical screen. The companion app obtains this via `overlayView.getWindowVisibleDisplayFrame(rect)`, where `rect.top` gives the status bar height (the main vertical offset on most devices) and `rect.left` gives the horizontal offset (typically 0).

2. **`dpr`** (display pixel ratio) — `DisplayMetrics.density`, which equals the WebView's `window.devicePixelRatio` on Android. This converts physical pixels to CSS pixels.

The formula (implemented in `onMessage()` in §2.1):

```
clientX = (touchPoint.x − webViewOffsetX) / dpr
clientY = (touchPoint.y − webViewOffsetY) / dpr
```

These `clientX/Y` values are then passed directly into the `PointerEvent` constructor. `PointerEvent.clientX/Y` is defined as viewport-relative CSS pixels, which is exactly what Excalidraw's hit-testing and coordinate mapping expect.

**Why not compute the offset in JavaScript?** `window.screenX/screenY` is not reliably populated in Android WebView, and there is no JavaScript API that exposes the WebView window's position on the physical screen. The companion app (running in the same Android process that manages windows) can obtain this reliably, which is why the offset travels in the wire protocol rather than being computed client-side.

**Orientation changes:** When the device rotates, `webViewOffsetX/Y` and `dpr` may change. The companion app should re-query these values in its `onLayoutChange` listener and include the updated values in subsequent events.

#### 2.4 Eraser Tool Handling

When `eraser_begin` is received:
1. Before dispatching the `pointerdown` event, programmatically switch Excalidraw to the eraser tool via the imperative API:
   ```typescript
   this.excalidrawAPI.setActiveTool({ type: "eraser" });
   ```
2. Dispatch the synthetic pointer events normally.
3. On `eraser_end`, switch back to the previously active tool.

#### 2.5 Pen Mode Activation

When the WebSocket connects successfully, automatically activate pen mode:
```typescript
this.excalidrawAPI.updateScene({
  appState: { penMode: true },
  captureUpdate: CaptureUpdateAction.NEVER,
});
```

This enables `penModeDoubleTapEraser` and `penModeSingleFingerPanning` behaviors.

---

### Phase 2.6: Stylus vs. Finger Input Separation

This is one of the most important design concerns: **how do we ensure that drawing with the stylus draws, while finger touches still pan the canvas and click buttons normally?**

#### How the Web Pointer Events API solves this natively

Every `PointerEvent` carries a `pointerType` field:

| Value | Meaning |
|---|---|
| `"pen"` | Physical stylus |
| `"touch"` | Finger or capacitive touch |
| `"mouse"` | Mouse or trackpad |

Excalidraw already reads this field. In pen mode, it draws when it sees `"pen"` pointer events and pans when it sees `"touch"` pointer events. All buttons and UI elements respond to any input type. **On a standard tablet that correctly reports input types, no extra work is needed.**

#### How the bridge preserves this separation

In this design, the two input streams are completely independent and arrive through different paths:

| Input | Path to WebView | `pointerType` seen by Excalidraw |
|---|---|---|
| Stylus | Onyx SDK → companion app → WebSocket → plugin → `dispatchEvent()` | `"pen"` (set explicitly in `BooxStylusReceiver`) |
| Finger / touch | Android normal touch dispatch → WebView directly | `"touch"` (set by the OS) |

Because `BooxStylusReceiver` explicitly sets `pointerType: "pen"` on every synthetic event it creates, and finger events arrive naturally from Android with `pointerType: "touch"`, Excalidraw sees exactly the right distinction. Clicking toolbar buttons, panning with one finger, and pinch-to-zoom all continue to work through the normal Android → WebView touch path and are completely unaffected by the companion app.

#### Role of `penModeSingleFingerPanning`

Activating pen mode (§2.5) also activates `penModeSingleFingerPanning`. This is the plugin setting that makes "finger = pan, stylus = draw" work: in pen mode, single-finger touch events are interpreted as panning gestures rather than drawing strokes. This means the user can:

- Draw with the stylus → Excalidraw draws (synthetic `"pen"` events)
- Pan with one finger → Excalidraw pans (real `"touch"` events + `penModeSingleFingerPanning`)
- Pinch-to-zoom with two fingers → Excalidraw zooms (real `"touch"` events)
- Tap any button → normal DOM click (real `"touch"` event)

#### Critical risk: `openRawDrawing()` may swallow finger touches

The only threat to this clean separation is if `openRawDrawing()` on the companion app's overlay causes the Onyx display controller to intercept **all** input — including finger touches — at the driver level, before they reach the Obsidian WebView.

**Mitigations (in order of preference):**

1. **`FLAG_NOT_TOUCHABLE` on the overlay `SurfaceView`** — the overlay window must be created with `WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE`. This tells Android to pass all touch events through to the window behind (Obsidian), while the Onyx SDK's pen-specific raw capture (which operates below the Android input stack) continues to work.

2. **Gate `setRawDrawingEnabled` on pen presence only** — call `touchHelper.setRawDrawingEnabled(true)` only inside `onPenActive()` and clear it in `onEndRawDrawing()`. This minimizes the window during which any touch routing side-effects could occur.

3. **Fallback: skip `openRawDrawing()`, use input-reader only** — if `openRawDrawing()` cannot be used without swallowing finger touches, call `setRawInputReaderEnable(true)` without `openRawDrawing()`. This loses the driver-level event capture that eliminates the Android input dispatch delay (~50–100ms), but preserves normal finger routing.

---

### Phase 3: Settings UI

Add a new section "Boox Stylus" to the plugin settings panel (`src/core/settings.ts` and the settings tab):

| Setting | Type | Default | Description |
|---|---|---|---|
| `booxStylusEnabled` | boolean | `false` | Enable Boox stylus bridge via WebSocket |
| `booxStylusPort` | number | `7854` | Local port for the companion app WebSocket |
| `booxStylusAutoConnect` | boolean | `true` | Auto-connect when a drawing is opened |
| `booxStylusActivatePenMode` | boolean | `true` | Automatically activate pen mode on connect |
| `booxStylusDebugLog` | boolean | `false` | Log stylus events to console for debugging |

Add to `ExcalidrawSettings` interface and `DEFAULT_SETTINGS` object.

---

### Phase 4: E-ink Display Optimization

Even after correct event forwarding, the default Excalidraw rendering will still feel laggy on an E-ink display because:
- E-ink refresh rate is much lower than LCD/AMOLED.
- Full waveform refreshes (needed for high quality) are slow (~500ms).
- Partial "fast" refreshes are faster but leave ghosting.

**Optimization strategies:**

#### 4.1 Simplified Stroke Preview (Predictive Ink)

While a stroke is in progress, render only a lightweight preview on a separate `<canvas>` overlaid on top of Excalidraw. This canvas uses the fastest possible drawing (straight line segments from `TouchPointList`). When the stroke ends (`pointerup`), hide the preview canvas and let Excalidraw render the final spline with pressure modulation.

```typescript
// Overlay canvas for predictive ink
const previewCanvas = document.createElement("canvas");
previewCanvas.style.position = "absolute";
previewCanvas.style.pointerEvents = "none";
excalidrawContainer.appendChild(previewCanvas);
```

This way, the user sees _immediate_ visual feedback during drawing, even if Excalidraw's final render arrives one or two refresh cycles later.

#### 4.2 Reduce `simulatePressure`

For Boox, real pressure data is available. Disable `simulatePressure` in `PenOptions`:

```typescript
penOptions: {
  simulatePressure: false,
  constantPressure: false,
  ...
}
```

This reduces CPU work during stroke processing.

#### 4.3 `streamline` Tuning

Increase `streamline` in `StrokeOptions` (e.g. from 0.5 to 0.7) to reduce the number of re-renders triggered per stylus sample. This trades off real-time responsiveness for rendering cost – acceptable on E-ink where a re-render is expensive anyway.

#### 4.4 E-ink Specific Pen Preset

Add a new built-in pen type `"boox-eink"` to the `PENS` registry in `src/utils/pens.ts`:
- `constantPressure: false` (use real pressure)
- `simulatePressure: false`
- `thinning: 0.3` (less thinning variability → more consistent ink weight on E-ink)
- `streamline: 0.7` (smoother, fewer segment updates)
- `strokeWidth: 2` (thicker default – more visible on E-ink)
- Auto-apply when Boox stylus bridge connects.

---

### Phase 5: Testing and Validation

1. **Unit tests** for coordinate translation logic.
2. **Integration test harness**: a mock WebSocket server that replays recorded `TouchPointList` sessions to validate event dispatch without a physical Boox device.
3. **Manual test on device**: verify stroke accuracy, pressure response, eraser switch, palm rejection.
4. **Performance profiling**: measure Excalidraw render latency per stroke and tune `streamline`/`thinning` to minimize re-render cost.

---

## Track B: Native Obsidian Android Plugin Extension (Future)

This track would be viable if Obsidian adds a native Android plugin extension mechanism (e.g. via a JNI bridge or a JavaScript-to-Java `JavascriptInterface`).

In that scenario:
- A Kotlin class is loaded directly inside the Obsidian process.
- `TouchHelper` is initialized against the WebView's own `SurfaceView`.
- Events are dispatched directly via `JavascriptInterface` without a WebSocket.
- Coordinates are naturally in WebView-space, eliminating the translation problem.

Until Obsidian exposes such an API, Track A remains the only viable path.

---

## Implementation Checklist

### Companion Android App
- [ ] Create new Android project with Onyx SDK dependencies
- [ ] Implement `Application` class with `RxManager` init and `HiddenApiBypass`
- [ ] Implement `StylusBridgeService` with `TouchHelper` and transparent overlay
- [ ] Implement `RawInputCallback` forwarding events to WebSocket
- [ ] Implement local WebSocket server (port 7854, localhost-only)
- [ ] Define JSON wire protocol for `StylusEvent`
- [ ] Implement `MainActivity` toggle (start/stop service)
- [ ] Implement `TileService` for Quick Settings
- [ ] Add foreground notification with Stop action
- [ ] Handle device rotation (`onLayoutChange` re-init)
- [ ] Test with Excalidraw plugin that events arrive correctly

### Plugin TypeScript Changes
- [ ] Add `booxStylusEnabled` and related settings to `ExcalidrawSettings`
- [ ] Add Boox Stylus section to settings UI
- [ ] Create `src/utils/booxStylusReceiver.ts` module
- [ ] Add `BooxStylusReceiver` lifecycle management in `ExcalidrawView.ts`
- [ ] Implement screen-to-canvas coordinate translation
- [ ] Implement eraser tool switching on eraser events
- [ ] Implement pen mode auto-activation on connect
- [ ] Add `"boox-eink"` pen preset to `src/utils/pens.ts`
- [ ] Add predictive ink overlay canvas
- [ ] Add connection status indicator in the Excalidraw toolbar
- [ ] Write unit tests for coordinate translation
- [ ] Write integration test with mock WebSocket server

---

## Open Questions

1. **Does `openRawDrawing()` suppress normal Android `MotionEvent` dispatch to the WebView?**  
   If yes, we may need to call `TouchHelper.setRawInputReaderEnable(true)` but *not* `openRawDrawing()`. This would give us stylus events but may lose some of the driver-level capture that avoids the 50–100ms Android input dispatch delay.

2. **What is the coordinate origin of `TouchPoint.x / .y`?**  
   Likely screen-absolute, but must be confirmed against the actual overlay view's position on the Boox device.

3. **Does the Onyx SDK provide hover events (pen near screen but not touching)?**  
   This would allow showing a cursor preview in Excalidraw before the stroke begins.

4. **What is the effective sampling rate of `TouchPointList`?**  
   The Onyx SDK typically captures at 100–200Hz. The WebSocket bridge will add some latency (~1–5ms on localhost). This should be acceptable.

5. **WebSocket security:** The local WebSocket binds to `127.0.0.1`. Is there a risk of other apps on the device connecting to it? On Android, apps are sandboxed – loopback connections from other apps are blocked unless the device is rooted. This should be safe, but worth noting in the companion app's documentation.

---

## Known Weaknesses and Risks

This section catalogs design weaknesses that must be resolved before the implementation is considered production-ready. Items marked 🔴 are **blockers** – they will cause incorrect behavior if unaddressed. Items marked 🟡 are important but not blocking for an initial working prototype.

### ✅ W1: Coordinate translation formula — **resolved**

~~The sample code in §2.1 contains a bug...~~

**Resolution (applied in §1.3 and §2.1):** The wire protocol now carries `webViewOffsetX`, `webViewOffsetY` (physical pixels; obtained from `overlayView.getWindowVisibleDisplayFrame()` in the companion app), and `dpr` (`DisplayMetrics.density`). The TypeScript receiver converts with:

```typescript
const clientX = (event.x - event.webViewOffsetX) / event.dpr;
const clientY = (event.y - event.webViewOffsetY) / event.dpr;
```

The companion app re-computes these on every orientation change via its `onLayoutChange` listener.

---

### ✅ W2: Auto-reconnect is broken — **resolved**

~~The `BooxStylusReceiver` in §2.1 has a dead reconnection path...~~

**Resolution (applied in §2.1):** `connect()` now sets `this.enabled = true` at the start, and `disconnect()` sets `this.enabled = false`. The `reconnect()` guard now executes correctly. Reconnection is attempted every 2 s after any unintentional close, and stops permanently only when `disconnect()` is called explicitly.

---

### ✅ W3: No `pointercancel` on WebSocket disconnect — **resolved**

~~If the WebSocket drops while a stroke is in progress...~~

**Resolution (applied in §2.1):** `BooxStylusReceiver` now tracks `strokeInProgress`. The `onclose` handler dispatches a synthetic `pointercancel` event on the canvas when a stroke is in progress, clearing Excalidraw's drawing state before attempting reconnection.

---

### 🟡 W4: WebSocket throughput at 200Hz

At 200 stylus samples per second, the bridge transmits 200 JSON-framed WebSocket messages per second. Each message has TCP+WebSocket framing overhead (~14–30 bytes) plus JSON serialization overhead. On localhost this is unlikely to saturate bandwidth, but JSON parsing in the JavaScript `onmessage` handler runs on the main thread and may contribute 1–3ms of per-message latency.

**Mitigation options:**
- Use binary MessagePack encoding instead of JSON (requires a library on both sides).
- Batch multiple `TouchPoint`s from `onRawDrawingTouchPointListReceived` into a single WebSocket message with a `type: "batch"` wrapper.
- Accept the latency for a first version; profile before optimizing.

---

### 🟡 W5: No palm rejection

The design forwards all stylus events from `RawInputCallback` but does not address palm rejection (the user's palm resting on the screen while drawing). The Onyx SDK provides an `ExcludeRect` list in `TouchHelper.setLimitRect(bounds, excludeRects)` that can mark regions as "palm zones". Without configuring this, palm contact may generate spurious `pointerdown` events.

**Mitigation:** Initially rely on Excalidraw's existing pen mode behavior (which already ignores `"touch"` events during active pen strokes). For a more robust solution, configure Onyx `ExcludeRect` based on stylus proximity (if hover events are available) to dynamically mark the palm zone.

---

### 🟡 W6: DPI-unaware canvas selector

The code in §2.2 selects the interactive canvas with:
```typescript
this.excalidrawContainer?.querySelector("canvas.interactive")
```

Excalidraw uses a CSS class on the interactive canvas layer, but this is an internal implementation detail that could change in an upstream Excalidraw update. A more resilient approach is to find the canvas by its role in the DOM or to use the `onPointerUpdate` API callback that Excalidraw already exposes via `ExcalidrawImperativeAPI`.

---

### 🟡 W7: `SYSTEM_ALERT_WINDOW` permission friction

The companion app requires `SYSTEM_ALERT_WINDOW` (overlay permission) to display a transparent `SurfaceView` over Obsidian. On Android 6+, this is not a standard install-time permission — users must grant it manually via **Settings → Apps → Special app access → Display over other apps**. This is a significant usability hurdle for non-technical users and must be clearly communicated with a setup guide.

---

### 🟡 W8: Battery and lifecycle management

A foreground service running `TouchHelper` continuously will prevent the CPU from going to a low-power state while Obsidian is in the background. This is particularly noticeable on E-ink devices where battery life is a priority.

**Resolution:** The companion app should monitor Obsidian's foreground state (e.g., via an `ActivityLifecycleCallbacks` broadcast or by the plugin sending a WebSocket "app paused/resumed" message) and pause `TouchHelper` when Obsidian is not visible. Alternatively, expose a manual pause button in the Quick Settings tile.

---

### 🟡 W9: Onyx SDK version fragility

The `TouchHelper.create()` API changed signature between Onyx SDK 1.3.x and 1.4.x, and the callbacks available in `RawInputCallback` vary by firmware version (e.g., `onRawDrawingTouchPointListReceived` was added in a later release). Pinning to `onyxsdk-pen:1.4.11` may fail on Boox devices running older firmware that ships an older bundled SDK.

**Resolution:** Target the lowest common denominator callback set (`onBeginRawDrawing`, `onRawDrawingTouchPointMoveReceived`, `onEndRawDrawing`) which have existed since 1.3.x, and treat `onRawDrawingTouchPointListReceived` as an optional enhancement detected at runtime via `try/catch` or API version check.

---

### ✅ W10: No `pointerId` for eraser end vs. pen end — **resolved**

~~The bridge uses a fixed `pointerId: 1` for all synthetic events...~~

**Resolution (applied in §2.1):** Pen-tip events use `pointerId: 1` (constant `PEN_POINTER_ID`) and eraser events use `pointerId: 2` (constant `ERASER_POINTER_ID`). Excalidraw's internal tracking correctly associates strokes with their respective pointer IDs.

---

## Overall Readiness Assessment

**The plan is now ready to execute.** The three blockers identified in the initial weakness analysis have been resolved in the design:

1. **W1 (coordinate translation)** — resolved. The wire protocol is extended with `webViewOffsetX`, `webViewOffsetY`, and `dpr`, computed by the companion app from `getWindowVisibleDisplayFrame()` and `DisplayMetrics.density`. The TypeScript receiver applies the correct formula: `clientX = (x − webViewOffsetX) / dpr`. See §1.3 and §2.3.

2. **W2 (dead auto-reconnect)** — resolved. `BooxStylusReceiver.connect()` now sets `enabled = true`, and `disconnect()` sets `enabled = false`, making the reconnect path live. See §2.1.

3. **W3 (no `pointercancel` on disconnect)** — resolved. The `onclose` handler now dispatches a `pointercancel` event when a stroke is in progress, using the `strokeInProgress` flag. See §2.1.

Additionally, **W10 (eraser `pointerId`)** was fixed as a zero-cost improvement in the same code block: the pen tip uses `pointerId: 1` and the eraser end uses `pointerId: 2`.

**Remaining items (W4–W9)** are performance, polish, and usability concerns that are appropriate to address in a second iteration after a working prototype is validated on a physical Boox device:

| Item | Severity | Recommended timing |
|---|---|---|
| W4: 200Hz JSON throughput | 🟡 | Profile after first prototype; batch if needed |
| W5: Palm rejection | 🟡 | Second iteration |
| W6: Fragile canvas selector | 🟡 | Fix if upstream Excalidraw changes break it |
| W7: SYSTEM_ALERT_WINDOW UX | 🟡 | Document in README before any release |
| W8: Battery/lifecycle | 🟡 | Second iteration — add pause/resume messages |
| W9: Onyx SDK version fragility | 🟡 | Limit callbacks to 1.3.x baseline before shipping |

---

## References

- [`boox-stylus-api-research.md`](./boox-stylus-api-research.md) – API documentation
- Onyx SDK developer portal: https://developer.onyx.net (requires registration)
- `@zsviczian/excalidraw` Pointer Events: see `onPointerUpdate` in `ExcalidrawView.ts:4301`
- Existing pen mode settings: `src/core/settings.ts:102–105`
- Existing pen style infrastructure: `src/utils/pens.ts`, `src/types/penTypes.ts`
- `ExcalidrawView.ts` Excalidraw React element props: lines 6222–6270
