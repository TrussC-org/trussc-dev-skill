# TrussC App Lifecycle & Utilities

## App Class

App inherits from RectNode. Entry point:

```cpp
int main() {
    WindowSettings settings;
    settings.setSize(1280, 720).setTitle("My App");
    TC_RUN_APP(MyApp, settings);
}

class MyApp : public App {
    void setup() override { }
    void update() override { }
    void draw() override { }
};
```

`TC_RUN_APP` is the universal entry point macro. It automatically selects between static linking and hot reload mode.

### Hot Reload

Edit source files and see changes live without restarting the app. Supported on macOS (.dylib) and Linux (.so).

**Enable:** Add `TC_HOT_RELOAD(YourAppClass)` at the top of your App's `.cpp` file:

```cpp
// tcApp.cpp
#include "tcApp.h"
TC_HOT_RELOAD(tcApp)

void tcApp::setup() { ... }
void tcApp::draw() { ... }
```

**How it works:**
1. CMake detects `TC_HOT_RELOAD` in `src/*.cpp` during configure
2. Build splits into Host (EXE) + Guest (shared library)
3. Host watches `src/` for file changes → auto-rebuilds Guest → reloads via dlopen
4. App state resets on reload (setup() is called again)

**Disable:** Comment out or remove `TC_HOT_RELOAD(...)` → next configure reverts to single-binary.

**Limitations:**
- State is not preserved across reloads (setup() re-runs)
- Not supported on Windows, Wasm, iOS, Android (falls back to static mode)
- `main.cpp` changes require a full restart

**When to use:** During iterative development — tweaking visuals, layout, animations, shader parameters. Saves significant time compared to full restart cycles.

### Virtual Methods

```cpp
// Lifecycle
virtual void setup()
virtual void update()
virtual void draw()
virtual void cleanup()
virtual void exit()                    // Before cleanup, save settings here

// Keyboard
virtual void keyPressed(int key)
virtual void keyReleased(int key)

// Mouse (screen coordinates)
virtual void mousePressed(Vec2 pos, int button)
virtual void mouseReleased(Vec2 pos, int button)
virtual void mouseMoved(Vec2 pos)
virtual void mouseDragged(Vec2 pos, int button)
virtual void mouseScrolled(Vec2 delta)

// Window / IO
virtual void windowResized(int width, int height)
virtual void filesDropped(const vector<string>& files)
```

Events dispatch to both App virtual methods AND the node scene graph.

---

## Window Management

### WindowSettings

```cpp
WindowSettings settings;
settings.width = 1280;
settings.height = 720;
settings.title = "My App";
settings.highDpi = true;           // Retina support
settings.pixelPerfect = false;     // Logical coords vs framebuffer pixels
settings.sampleCount = 4;          // MSAA
settings.fullscreen = false;
```

### Runtime

```cpp
setWindowTitle(string)
setWindowSize(int w, int h)
setFullscreen(bool); toggleFullscreen(); isFullscreen()
getWindowWidth(); getWindowHeight(); getWindowSize() -> Vec2
getAspectRatio()
getDpiScale()                      // 2.0 on Retina

// Window position (macOS/Windows only; others return IVec2(-1,-1))
getWindowPosition() -> IVec2       // Screen coords, top-left origin
setWindowPosition(int x, int y)
```

---

## Frame Timing

### FPS Control

```cpp
setFps(fps)                        // Sync update/draw
setIndependentFps(updateFps, drawFps)  // Independent rates

// Special constants
VSYNC   = -1.0f    // Match monitor refresh
EVENT_DRIVEN = 0.0f // Only on redraw()

redraw(count = 1)                  // Trigger draw frames
```

**Typical pattern for apps with UI:**
```cpp
setIndependentFps(VSYNC, 0);      // Update at vsync, draw only on redraw()
```

### Timing Queries

```cpp
getElapsedTime()                   // Seconds since start
getDeltaTime()                     // Seconds since last update() call (real elapsed)
getFrameRate()                     // FPS based on update frequency (10-frame average)
getFrameCount()                    // Total update() calls
getUpdateCount()
getDrawCount()
```

---

## Input State

### Mouse

```cpp
getGlobalMouseX(); getGlobalMouseY()
getGlobalPMouseX(); getGlobalPMouseY()  // Previous frame
getMousePos() -> Vec2
isMousePressed()
getMouseButton()                   // -1 if none
```

### Keyboard

```cpp
isKeyPressed(key)                  // Is key currently held? (exact key, e.g. KEY_LEFT_SHIFT vs KEY_RIGHT_SHIFT distinct)

// "Either-side" modifier helpers — return true if either left or right variant is held
isShiftPressed()                   // KEY_LEFT_SHIFT || KEY_RIGHT_SHIFT
isControlPressed()                 // KEY_LEFT_CONTROL || KEY_RIGHT_CONTROL
isAltPressed()                     // KEY_LEFT_ALT || KEY_RIGHT_ALT (Option on macOS)
isSuperPressed()                   // KEY_LEFT_SUPER || KEY_RIGHT_SUPER (Cmd on macOS, Win on Windows)
```

Safe to call from any thread-relevant entry point (setup / update / draw / event handlers) — these are just lookups into the global key-state set.

**IMPORTANT: Letter keys are UPPERCASE** — sokol uses key codes, not ASCII characters.
```cpp
// CORRECT
if (key == 'A') { /* ... */ }

// WRONG — will never match
if (key == 'a') { /* ... */ }
```

**Common pattern — modifier + key combo:**
```cpp
void keyPressed(int key) {
    if (isShiftPressed()) {
        switch (key) {
            case KEY_LEFT:  /* shift + left  */ break;
            case KEY_RIGHT: /* shift + right */ break;
        }
    } else {
        // plain arrow keys
    }
}
```
The modifier helper works inside `keyPressed()` because the shift key event already updated `keysPressed` before the arrow key event fires.

Key constants:
```
KEY_SPACE, KEY_ESCAPE, KEY_ENTER, KEY_TAB, KEY_BACKSPACE, KEY_DELETE
KEY_LEFT, KEY_RIGHT, KEY_UP, KEY_DOWN
KEY_F1 ... KEY_F12
KEY_LEFT_SHIFT, KEY_RIGHT_SHIFT, KEY_LEFT_CONTROL, KEY_LEFT_ALT, KEY_LEFT_SUPER
MOUSE_BUTTON_LEFT (0), MOUSE_BUTTON_RIGHT (1), MOUSE_BUTTON_MIDDLE (2)
```

---

## Logging

```cpp
logNotice()  << "info message";
logWarning() << "warning message";
logError()   << "error message";

logNotice("TAG") << "tagged message";
```

**Never use cout** — stdout is reserved for MCP communication. Logs go to stderr.

LogLevel: Verbose, Notice, Warning, Error, Fatal, Silent

---

## Headless verification (screenshot a running app)

To check what a GUI app actually renders — in CI, over SSH, or from an agent —
drive the app's own MCP server over HTTP. It encodes the app's framebuffer to
PNG internally, so it captures exactly the app window and needs **no
screen-recording / TCC permission** (unlike macOS `screencapture`, which fails
on locked-down terminals and grabs other windows too).

```bash
BIN=bin/MyApp.app/Contents/MacOS/MyApp        # the real binary, not `open`
TRUSSC_MCP=1 TRUSSC_MCP_PORT=8765 "$BIN" >/tmp/app.log 2>&1 &
sleep 3                                        # wait for "MCP HTTP server started"

URL=http://localhost:8765/mcp                  # NOTE: path is /mcp, not /
curl -s -X POST $URL -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{}}'
curl -s -X POST $URL -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":2,
       "params":{"name":"save_screenshot","arguments":{"path":"/tmp/shot.png"}}}'
```

Then open `/tmp/shot.png`. Gotchas:

- Endpoint is `http://localhost:PORT/mcp` — posting to `/` returns **404**.
- `save_screenshot {path}` writes a file (best for this flow). `get_screenshot`
  returns base64 PNG inline; a per-frame PNG encode can contend with it, so
  prefer `save_screenshot`.
- Standard MCP tools when `TRUSSC_MCP=1`: `mouse_move`, `mouse_click`,
  `key_press`, `get_screenshot`, `save_screenshot` (+ any app-specific tools).
  So you can also *script interaction* before capturing.
- In-app alternative for one-off captures: `fbo.save("shot.png")` (see
  graphics.md) or render a frame and save the default framebuffer.

This is the reliable path when the GUI/MCP launch itself is flaky — verify by
framebuffer, not by what's on the physical screen.

---

## Math

### Constants

```cpp
TAU          // 6.283... (full rotation, 2π) — USE THIS
HALF_TAU     // π
QUARTER_TAU  // π/2
PI           // deprecated, use HALF_TAU
```

### Vectors

```cpp
Vec2(x, y)
Vec3(x, y, z)
Vec4(x, y, z, w)

// Integer vectors (for pixel coords, grid positions, etc.)
IVec2(x, y)                        // int x, y
IVec3(x, y, z)                     // int x, y, z
iv.toVec2()                        // Convert to float Vec2
iv.toVec3()                        // Convert to float Vec3
iv.xy()                            // IVec3 → IVec2

// Vec2/Vec3 methods
v.length(); v.lengthSquared()
v.normalized(); v.normalize()      // normalize() mutates
v.dot(other)
v.distance(other)
v.lerp(target, t)
v.angle()                         // Radians from +x (Vec2)
v.rotated(radians)                // Vec2
v.perpendicular()                 // Vec2

Vec2::fromAngle(radians, length)  // Static constructor
```

### Utility Functions

```cpp
lerp(a, b, t)
clamp(value, min, max)
map(value, inMin, inMax, outMin, outMax)
radians(degrees)
degrees(radians)
```

---

## Threading

### Thread Class

```cpp
class MyWorker : public Thread {
    void threadedFunction() override {
        while (isThreadRunning()) {
            // do work
            sleep(100);  // milliseconds
        }
    }
};

MyWorker worker;
worker.startThread();
worker.stopThread();       // Signal stop
worker.waitForThread();    // Block until done
worker.isThreadRunning();

Thread::isCurrentThreadTheMainThread();
Thread::sleep(milliseconds);
```

### Thread Safety Rules

1. **Never sleep inside a mutex lock** — deadlock
2. **Use lock_guard<mutex>** for shared data
3. **Pixels OK in background** — Image/Texture/Fbo are main-thread only
4. **Use ThreadChannel** for safe cross-thread message passing

### Timers (frame + async)

Any Node (the App is one) can schedule callbacks. `callAfter`/`callEvery` fire
from the **update loop** — main-thread and simple, but quantized to the frame
rate (~16 ms) and slightly drifty. `callAfterAsync`/`callEveryAsync` fire from a
**precise background scheduler thread** — no frame jitter, no drift — for
sequencer clocks, LED/MIDI output, anything timing-sensitive.

```cpp
uint64_t a = callEvery(0.5, [this]{ tick(); });        // frame-driven
cancelTimer(a); cancelAllTimers();

uint64_t b = callEveryAsync(0.14, [this]{ step(); });  // off-thread, precise
cancelAsyncTimer(b); cancelAllAsyncTimers();           // e.g. on mode change
```

The async callback runs **on the scheduler thread**, so the same rules as audio
callbacks apply — guard shared state with a mutex, never draw from it,
`AudioEngine::play()` is fine. **Gotcha:** call `cancelAsyncTimer` /
`cancelAllAsyncTimers` WITHOUT holding the callback's mutex — cancel waits for an
in-flight callback, which needs that mutex, so holding it deadlocks. `~Node`
cancels leftovers and waits in-flight. Async is **native only** (real thread).

### Real-time audio & MIDI callbacks

Two engine callbacks run on their **own** threads, not the update/draw thread.

**`AudioEngine::audioOut`** fires on the audio thread, once per device buffer.
Listeners take an ordering priority (`tc::audio::priority`):

- `Generator = 100` (default) — a synth: **ADD** your samples into `b.data`.
- `Effect = 500` — reads + writes the buffer (filter / reverb).
- `Monitor = 900` — **read-only, runs last**: scope / FFT / recorder.

Inside the callback do **only** cheap work — no allocation, no locks, no
`AudioEngine::play()/load()`. For an oscilloscope, copy into a lock-free ring
and draw it from the main thread:

```cpp
std::array<float, 1024> ring_{}; std::atomic<int> wpos_{0};
EventListener tap_ = AudioEngine::getInstance().audioOut.listen(
    [this](AudioOutBuffer& b){                       // audio thread
        int w = wpos_.load(std::memory_order_relaxed);
        for (int i = 0; i < b.frameCount; ++i) { ring_[w] = b.data[i*b.channels]; w = (w+1)%1024; }
        wpos_.store(w, std::memory_order_release);
    }, audio::priority::Monitor);
```

**`MidiIn::onMessage`** (tcxMidi) fires on libremidi's input thread the instant
a message arrives — **event-driven, far lower jitter than polling**
`getNextMessage()` once per frame. Two facts make it safe to trigger sound right
there:

- `AudioEngine::play()` is internally mutex-guarded (same lock as the mixer), so
  it's safe to call from the MIDI thread.
- The audioOut listener never touches your app mutex, so there's no lock
  inversion with the audio thread.

Guard any state the callback shares with draw()/update() behind a `std::mutex`,
and keep the returned `EventListener` alive (RAII — dropping it unsubscribes):

```cpp
EventListener midiTap_ = midiIn_.onMessage.listen([this](MidiMessage& m){
    std::lock_guard<std::mutex> lock(mtx_);   // guards synth + visual state
    if (m.isNoteOn()) voice_.play();          // play() is thread-safe
});
```

---

## 3D Projection

```cpp
setupScreenOrtho()                 // 2D mode
setupScreenPerspective(fovDeg=45)  // 3D
setupScreenFov(fovDeg)             // 0=ortho, >0=perspective

setNearClip(float); setFarClip(float)

worldToScreen(Vec3) -> Vec3        // (screenX, screenY, depth)
screenToWorld(Vec2, worldZ=0) -> Vec3
```

---

## Tween (Standalone Animation)

Animate any type that supports lerp (float, Vec2, Vec3, Color, etc.). Auto-updates via `events().update` — no manual update needed.

```cpp
Tween<float> tween;
tween.from(0).to(100).duration(1.0)
     .ease(EaseType::Cubic, EaseMode::InOut)
     .delay(0.5)           // Wait before starting
     .start();

float val = tween.getValue();      // Current interpolated value
float progress = tween.getProgress(); // 0.0–1.0
```

### Chainable Setters

```cpp
.from(T value)                     // Start value
.to(T value)                       // End value
.duration(float seconds)
.ease(EaseType, EaseMode)          // EaseType: Linear, Quad, Cubic, Quart, Expo, Sine, Elastic, Bounce, Back
.ease(EaseType in, EaseType out)   // Asymmetric easing
.delay(float seconds)              // Delay before animation (re-applied each loop)
.loop(int count = -1)              // -1=infinite, N=repeat N times
.yoyo(bool = true)                 // Reverse direction each loop
.start()                           // Begin animation
.pause() / .resume() / .reset()
.finish()                          // Jump to end immediately
```

### Completion Event

```cpp
tween.complete->listen([]() {
    // Fires when all loops are done
});
```

### Loop + Delay + Yoyo

```cpp
// Move-wait-reverse-wait pattern
tween.from(0).to(200).duration(0.5)
     .delay(0.3)
     .loop(-1).yoyo()
     .start();
```

**Note:** For animating Node properties (position, scale, rotation), use TweenMod instead (see node-system.md).

---

## File Utilities

```cpp
// Data path (relative to exe + data root)
setDataPathRoot(path)
getDataPath(filename)
setDataPathToResources()           // macOS bundle Resources/

// Path parsing
getFileName(path)                  // "dir/test.txt" → "test.txt"
getBaseName(path)                  // "dir/test.txt" → "test"
getFileExtension(path)             // "dir/test.txt" → "txt"
getParentDirectory(path)

// Filesystem
fileExists(path)
directoryExists(path)
createDirectory(path)              // Creates parents too
listDirectory(path)                // vector<string>
joinPath(dir, file)
getAbsolutePath(path)
```

---

## String Utilities

```cpp
toString(value)
toString(value, precision)
toInt(str); toInt64(str)
toFloat(str); toDouble(str)
toBool(str)                        // "true"/"1"/"yes" → true
toHex(value); toBinary(value)
```
