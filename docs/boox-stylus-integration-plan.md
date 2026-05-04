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
    val x: Float,
    val y: Float,
    val pressure: Float,       // 0.0–1.0
    val tiltX: Float,
    val tiltY: Float,
    val timestamp: Long,
)
```

JSON wire format example:
```json
{
  "type": "pointermove",
  "x": 412.5,
  "y": 803.2,
  "pressure": 0.65,
  "tiltX": 12.0,
  "tiltY": -5.0,
  "timestamp": 1714850000123
}
```

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

export class BooxStylusReceiver {
  private ws: WebSocket | null = null;
  private canvas: HTMLCanvasElement | null = null;
  private enabled: boolean = false;

  connect(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.ws = new WebSocket("ws://127.0.0.1:7854");
    this.ws.onmessage = (evt) => this.onMessage(JSON.parse(evt.data));
    this.ws.onclose = () => setTimeout(() => this.reconnect(), 2000);
  }

  private reconnect() {
    if (this.enabled) this.connect(this.canvas!);
  }

  private onMessage(event: StylusEventMessage) {
    if (!this.canvas) return;
    const pt = this.canvas.getBoundingClientRect();
    // Coordinates from the overlay are screen-absolute; adjust to canvas-relative
    const clientX = event.x - pt.left + this.canvas.offsetLeft;
    const clientY = event.y - pt.top + this.canvas.offsetTop;

    // Map event type to PointerEvent type
    const pointerEventType = pointerEventTypeMap[event.type];
    if (!pointerEventType) return;

    const syntheticEvent = new PointerEvent(pointerEventType, {
      bubbles: true,
      cancelable: true,
      pointerType: "pen",
      pointerId: 1,
      clientX,
      clientY,
      pressure: event.pressure,
      tiltX: event.tiltX,
      tiltY: event.tiltY,
    });
    this.canvas.dispatchEvent(syntheticEvent);
  }

  disconnect() {
    this.ws?.close();
    this.ws = null;
  }
}

const pointerEventTypeMap: Record<string, string> = {
  pointerdown: "pointerdown",
  pointermove: "pointermove",
  pointerup: "pointerup",
  eraser_begin: "pointerdown",   // switch tool to eraser first
  eraser_move: "pointermove",
  eraser_end: "pointerup",
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

The companion app's `TouchPoint` coordinates are **screen-absolute** (full device screen pixels). The Excalidraw canvas sits inside:

```
Android screen
  └── Obsidian app window
        └── WebView (may have status bar offset)
              └── Obsidian workspace
                    └── Excalidraw leaf pane
                          └── <canvas> element
```

The bridge must account for this offset chain. Two options:

**Option A (Simpler):** The companion app reads the `WindowManager` `Display` dimensions and the `SurfaceView` position to compute the absolute offset of the canvas within the screen, then sends coordinates relative to the canvas. This requires the companion app to be aware of the canvas bounds.

**Option B (More robust):** Send screen-absolute coordinates from the companion app; in the TypeScript receiver, use `element.getBoundingClientRect()` relative to the WebView's top-left to convert to canvas-relative client coordinates.

Option B is recommended because the JavaScript side can always query its own position, while the Android side cannot reliably know the WebView layout.

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

### 🔴 W1: Coordinate translation formula is incorrect

The sample code in §2.1 contains a bug:

```typescript
const clientX = event.x - pt.left + this.canvas.offsetLeft;
const clientY = event.y - pt.top + this.canvas.offsetTop;
```

`event.x / .y` are **screen-absolute physical pixels** from the Onyx SDK. `getBoundingClientRect()` returns the canvas position in **viewport CSS pixels**. These units are incompatible without two corrections:

1. **WebView screen offset**: the WebView's top-left corner in screen-absolute coordinates must be subtracted. There is no direct JavaScript API for this; it must either be sent from the companion app or estimated from `window.screen` data.
2. **Device pixel ratio scaling**: screen-absolute physical pixels must be divided by `window.devicePixelRatio` to convert to CSS pixels.

The correct formula is approximately:
```typescript
const dpr = window.devicePixelRatio;
const webViewOffsetX = /* provided by companion app or estimated */;
const webViewOffsetY = /* provided by companion app or estimated */;
const clientX = (event.x - webViewOffsetX) / dpr;
const clientY = (event.y - webViewOffsetY) / dpr;
```

**Resolution:** The companion app should broadcast the WebView window's screen-absolute position (obtainable via `WindowManager` + `getWindowVisibleDisplayFrame`) alongside stylus events, and the wire protocol must include `dpr`. Alternatively the `StylusEvent` coordinates can be pre-converted to CSS viewport coordinates server-side if the companion app knows the WebView geometry.

---

### 🔴 W2: Auto-reconnect is broken — `enabled` flag is never set

The `BooxStylusReceiver` in §2.1 has a dead reconnection path:

```typescript
private enabled: boolean = false;   // never set to true

private reconnect() {
  if (this.enabled) this.connect(this.canvas!);  // never executes
}
```

If the WebSocket closes (companion app stopped, device sleep), the plugin silently stops receiving events. Fix: set `this.enabled = true` inside `connect()` and `this.enabled = false` inside `disconnect()`.

---

### 🔴 W3: No `pointercancel` on WebSocket disconnect — Excalidraw gets stuck

If the WebSocket drops while a stroke is in progress (mid-`pointerdown`), no `pointerup` is ever dispatched. Excalidraw remains in drawing mode with a dangling stroke, and the canvas becomes unresponsive to further input until the user manually changes tools or reloads.

**Resolution:** In the WebSocket `onclose` handler, dispatch a synthetic `pointercancel` event on the canvas before attempting reconnection:
```typescript
this.ws.onclose = () => {
  if (this.strokeInProgress) {
    this.canvas?.dispatchEvent(new PointerEvent("pointercancel", {
      bubbles: true, pointerType: "pen", pointerId: 1,
    }));
    this.strokeInProgress = false;
  }
  setTimeout(() => this.reconnect(), 2000);
};
```

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

### 🟡 W10: No `pointerId` for eraser end vs. pen end

The bridge uses a fixed `pointerId: 1` for all synthetic events. Some styluses report the eraser end as a separate tool (`pointerType: "eraser"` in standard browsers). If the Onyx SDK provides eraser events (`onBeginRawErasing`, etc.), these should be dispatched with a distinct `pointerId` (e.g., `2`) to avoid confusing Excalidraw's internal event tracking, which associates strokes by `pointerId`.

---

## Overall Readiness Assessment

**The architecture is sound in concept, but the implementation plan is not yet ready to execute.** Two blockers (W1, W3) must be resolved in the design before writing any code:

1. **W1 (coordinate translation)** is the most critical. Without correct coordinates, every stroke will appear in the wrong place. This requires a concrete decision on how the companion app communicates the WebView's screen-absolute position to the TypeScript receiver, and the wire protocol must be extended to include `devicePixelRatio`.

2. **W3 (no `pointercancel` on disconnect)** will make the canvas routinely get stuck during development/testing when the companion app is restarted, making it difficult to iterate.

Once W1 and W3 are resolved:
- W2 (reconnect bug) is a 2-line fix.
- W4–W10 are performance and polish concerns that can be deferred to a second iteration.

The recommended next step is to write a concrete coordinate-translation spec (confirming `TouchPoint` origin via Onyx SDK documentation or device testing) and to extend the wire protocol accordingly, before any Kotlin or TypeScript code is written.

---

## References

- [`boox-stylus-api-research.md`](./boox-stylus-api-research.md) – API documentation
- Onyx SDK developer portal: https://developer.onyx.net (requires registration)
- `@zsviczian/excalidraw` Pointer Events: see `onPointerUpdate` in `ExcalidrawView.ts:4301`
- Existing pen mode settings: `src/core/settings.ts:102–105`
- Existing pen style infrastructure: `src/utils/pens.ts`, `src/types/penTypes.ts`
- `ExcalidrawView.ts` Excalidraw React element props: lines 6222–6270
