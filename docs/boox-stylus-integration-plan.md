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

## References

- [`boox-stylus-api-research.md`](./boox-stylus-api-research.md) – API documentation
- Onyx SDK developer portal: https://developer.onyx.net (requires registration)
- `@zsviczian/excalidraw` Pointer Events: see `onPointerUpdate` in `ExcalidrawView.ts:4301`
- Existing pen mode settings: `src/core/settings.ts:102–105`
- Existing pen style infrastructure: `src/utils/pens.ts`, `src/types/penTypes.ts`
- `ExcalidrawView.ts` Excalidraw React element props: lines 6222–6270
