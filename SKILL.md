---
name: trussc-dev
description: TrussC app development — Node/Mod architecture, API patterns, graphics, trusscli build workflow, runtime verification via MCP, common pitfalls. Use when writing, building, or verifying TrussC C++ apps.
---

# TrussC Development Guide

Comprehensive environment for building apps with TrussC. This skill covers the full
loop: **design → implement → build → run → verify**. See topic files for detail.

## Quick Reference Files

- [architecture.md](architecture.md) — **How to structure an app**: Node hierarchy as code separation, Mods for cross-cutting behavior. Read before building anything beyond a single-file sketch.
- [node-system.md](node-system.md) — Node, RectNode, ScrollContainer, events, timers, mods (API reference)
- [graphics.md](graphics.md) — Drawing, colors, style, Image/Pixels/Texture/Fbo
- [app-lifecycle.md](app-lifecycle.md) — App setup, window, input, logging, threading, math
- [verify.md](verify.md) — **Runtime verification**: MCP screenshots, input injection, ImGui driving, evidence rules
- [addon-authoring.md](addon-authoring.md) — Creating & publishing addons (only when authoring one)

## The Development Loop

1. **Design** — structure the app as Node subclasses + Mods ([architecture.md](architecture.md)), not as one giant `tcApp::draw()`.
2. **Implement** — follow the patterns below. New/renamed/deleted source files → `trusscli update`.
3. **Build** — `trusscli build`. Fix errors before claiming anything.
4. **Verify** — launch with `TRUSSC_MCP=1`, screenshot, drive inputs, check state ([verify.md](verify.md)).
5. **Report** — verdict is **COMPLETE / COMPLETE WITH NOTES / BLOCKED** with an evidence table. A successful build alone is never COMPLETE.

## Build Workflow

### New Source File → re-update

When you create a new `.cpp` or `.h` file under `src/`, you MUST run:

```bash
trusscli update
```

TrussC uses `file(GLOB_RECURSE CONFIGURE_DEPENDS ...)`. Ninja picks up new files automatically, but Xcode does not. Always run `trusscli update` after adding/deleting/renaming source files.

### Build & Run

```bash
trusscli build                          # Build only
trusscli run                            # Build and launch
```

Always use `trusscli` instead of calling `cmake` directly. If `trusscli` can't do something you need, report it to the user.

### trusscli

If `CMakeLists.txt` or `CMakePresets.json` need regeneration (e.g., adding/removing addons):

```bash
trusscli update -p /path/to/project
```

Other common commands:
```bash
trusscli new /path/to/project           # Create new project
trusscli build                          # Build (native platform)
trusscli build --release                # Release config
trusscli run                            # Build and launch
trusscli clean                          # Delete build dirs
trusscli upgrade                        # Pull latest TrussC + rebuild trusscli
trusscli doctor                         # Check dev environment
trusscli info                           # Show project / framework info
```

## Addons (using them in your app)

Addons live in `addons/` next to the project. Before writing functionality from
scratch (networking, physics, CV, model loading…), **check whether an addon already
covers it.**

**Bundled with the framework** (in the TrussC repo's `addons/`):
tcxBox2d (2D physics), tcxCurl (HTTPS), tcxDepthCamera + tcxDepthRecord (depth cams),
tcxGltf / tcxObj (3D models), tcxHap (GPU video codec), tcxImGui (Dear ImGui),
tcxLua (scripting), tcxLut (color grading), tcxMidi (MIDI I/O),
tcxNodeInspector (runtime hierarchy + inspector + gizmo — Unity-style debug panel),
tcxOsc (OSC), tcxQuadWarp (projection mapping), tcxTls (TLS/SSL), tcxWebSocket.

**Community / registry** (cloned on demand): tcxArtnet (DMX/Art-Net), tcxAruco
(marker tracking), tcxAzureKinect, tcxGPT (OpenAI API), tcxGlitch (databending),
tcxIME (non-Latin text input), tcxMQTT, tcxOpenCV, tcxPhysics (Jolt 3D physics),
tcxPly (point clouds/meshes), tcxSyphon (macOS texture sharing) — and growing; the
list above goes stale, so query the registry for current truth.

### Querying the latest registry

The registry is a single JSON on the `trussc-addons` repo's gh-pages branch,
auto-rebuilt by a crawler (GitHub topic `trussc-addon`). `trusscli addon list
--remote` / `search` wrap it; fetch it directly when you want full metadata:

```bash
REG=https://raw.githubusercontent.com/TrussC-org/trussc-addons/gh-pages/registry.json

curl -s $REG | jq -r '.last_updated'                       # freshness stamp
curl -s $REG | jq -r '.addons[] | "\(.name)\t\(.description)"'
curl -s $REG | jq -r '.addons[] | select(.bundled|not) | .name'   # community only
curl -s $REG | jq '.addons[] | select(.category=="network")'
curl -s $REG | jq '.addons[] | select((.keywords|join(" "))+.description | test("midi";"i"))'
```

Per-addon fields: `name`, `owner`, `url`, `bundled`, `description`, `author`,
`license`, `category`, `keywords`, `platforms`, `screenshot`, `demo_url`,
`dependencies`, `stars`. Human-browsable version: https://trussc.org/addons/

```bash
trusscli addon list                     # Local addons + project status
trusscli addon list --remote            # Include registry (community)
trusscli addon search <query>           # Case-insensitive search of registry
trusscli addon add <addon>              # Add to project's addons.make
trusscli addon remove <addon>           # Remove from addons.make
trusscli addon clone <name>             # Clone from registry into addons/
trusscli addon clone <owner>/<name>     # Exact GitHub repo (disambiguates collisions)
trusscli addon clone <git-url>          # Direct HTTPS or git@host:user/repo SSH
trusscli addon pull --all               # git pull all cloned addons
```

Ambiguous bare names (e.g. a bundled `tcxLua` colliding with a community `funatsufumiya/tcxLua`) require `owner/name` form. Tab completion lists both forms.

**Writing your own addon?** → [addon-authoring.md](addon-authoring.md)

## Essential Patterns

### Namespace & Includes

```cpp
#include <TrussC.h>          // Angle brackets, not quotes
using namespace std;
using namespace tc;
// Omit std:: and tc:: prefixes everywhere
```

### Units & Constants

- **Angles:** TAU (= 2π, full rotation). NOT PI.
- **Colors:** 0.0–1.0 float range (not 0–255)
- **Logging:** `logNotice()`, `logWarning()`, `logError()` — NOT cout (stdout reserved for MCP)

### Event-Driven Rendering (redraw) — the GUI-app recipe

```cpp
setIndependentFps(60, EVENT_DRIVEN);  // update at a fixed 60, draw only on redraw()
```

This is the recommended mode for GUI/tool apps: a fixed update rate keeps timers,
tweens, and input polling deterministic regardless of monitor refresh (a 120 Hz
display won't double your update rate the way VSYNC does), while draw runs only
when something changed — an idle GUI costs near-zero GPU.

Call `redraw()` whenever display changes: key/mouse input, data updates, async load completion, window resize.

### Thread Safety

| Class | Main Thread | Background Thread |
|-------|:-----------:|:-----------------:|
| Pixels | OK | OK (CPU only) |
| Texture | OK | NO (GPU) |
| Image | OK | NO (GPU sync) |
| Fbo | OK | NO (GPU) |
| Font | OK | NO (GPU atlas) |

**Pattern:** Load `Pixels` in background thread → transfer to main thread → create Texture/Image.

### Node Hierarchy Basics

```cpp
auto child = make_shared<RectNode>();
child->setSize(100, 50);
child->setPos(10, 20);
parent->addChild(child);           // Automatic transform inheritance
child->enableEvents();             // Required for mouse events
child->setActive(false);           // Hides + disables update/draw/events
```

Draw order: `beginDraw()` → `draw()` → `drawChildren()` → `endDraw()`

Children draw in their parent's local coordinate space. No automatic clipping (use `setClipping(true)` on RectNode for scissor).

For *when and why* to split code into Nodes and Mods, see [architecture.md](architecture.md).

### Mouse Events

Override `onMousePress(Vec2 local, int button)` etc. Return `true` to consume (stop propagation).

Events are in **local coordinates** of the node. `enableEvents()` is required.

```cpp
bool onMousePress(Vec2 local, int button) override {
    if (button == 0) { /* left click */ return true; }
    return RectNode::onMousePress(local, button);  // propagate
}
```

### Timers

```cpp
uint64_t id = callAfter(0.5, [this]() { /* fires once after 0.5s */ });
uint64_t id2 = callEvery(1.0, [this]() { /* fires every 1s */ });
cancelTimer(id);
```

Protected Node methods. Store ID to cancel later.

## UI Widget Design Patterns

See [node-system.md](node-system.md) § "UI Widget Design Patterns" for details on:
- Every UI element as a RectNode child (labels, separators, buttons)
- Event<T>/EventListener RAII for widget communication (no raw `function<>` callbacks)
- PlainScrollContainer + ScrollBar + LayoutMod panel pattern
- Constructor/setup() separation (create in constructor, addChild in setup)
- Font rendering for all text

## Common Pitfalls

1. **Letter keys are UPPERCASE** → `'A'` not `'a'`. sokol uses key codes, not ASCII. `if (key == 'a')` will never match.
2. **sleep() inside mutex lock** → deadlock. Always sleep OUTSIDE lock scope.
3. **Image in background thread** → crash. Use Pixels for background, Image for main thread only.
4. **Texture update twice per frame** → second call silently skipped (sokol limitation).
5. **Forgot enableEvents()** → node won't receive mouse/key events.
6. **Forgot redraw()** → screen doesn't update in event-driven mode.
7. **setLineWidth()** → doesn't exist in sokol. drawLine is always 1px. For thick lines: `Path::drawStroke()` (uses StrokeMesh internally), or assemble StrokeMesh directly.
8. **std::map vs tc::map** → name collision with `using namespace tc`. Use `std::map` explicitly if needed, or avoid `using namespace tc` in that file.
9. **Copy Pixels/Image/Texture** → deleted copy constructor. Use `std::move()` or `Pixels::clone()`.
10. **Node without make_shared** → `addChild()` fails. Always create nodes with `make_shared<>()`.
11. **getGlobalPos()** → returns `Vec3` (origin of the node in global space). Shorthand for `localToGlobal(Vec3(0,0,0))`.
12. **removeChild() during iteration** → crashes (vector mutation). Use `destroy()` for deferred removal. Check `isDead()` to skip destroyed nodes.
13. **addChild() in constructor** → crash. `weak_from_this()` fails because `shared_ptr` isn't complete yet. Always `addChild()` in `setup()` override.
14. **Raw `function<>` callbacks** → no auto-cleanup, dangling risk. Use `Event<T>` + `EventListener` (RAII auto-disconnect on destruction).
15. **Drawing UI elements in parent draw()** → won't scroll, no hit testing. Make every UI element a RectNode child.
16. **LayoutMod auto-relayout** → doesn't happen. Call `updateLayout()` manually in `setSize()` and after child changes.
17. **IBL/`Environment` on iOS Safari (wasm)** → auto-skipped. The IBL bake renders into cube-face render targets, which iOS/iPadOS Safari (both WebGPU & WebGL2) can't do without breaking the canvas swapchain (one frame then black/frozen; loop keeps running). So `loadProcedural()`/`loadFromHDR()` no-op on iOS → PBR falls back to flat hemisphere ambient + direct lights. Plain 2D FBOs, normal maps, shadows are fine; native, desktop web and Android web bake IBL normally. Don't rely on IBL reflections for cross-platform looks if you target iOS web.
18. **setBlendMode() inside a 3D scene kills depth** → EVERY blend pipeline (Alpha, Add, even Disabled) has depth write/test OFF; only `internal::pipeline3d` (loaded by screen setup / `EasyCam::begin()`) writes depth. "Restoring" with `setBlendMode(Alpha)` after an additive effect leaves the rest of the scene depth-less — geometry renders in submission order, so a box's unlit BOTTOM face overwrites its lit front face (looks like a pitch-black face / lights mysteriously having no effect). Restore depth explicitly: `setBlendMode(BlendMode::Alpha); if (internal::pipeline3dInitialized) sgl_load_pipeline(internal::pipeline3d);`. See graphics.md § Blend Modes.
19. **EasyCam `enableMouseInput()` eats every left-press** → it consumes the press as orbit-start during `events().mousePressed`, and tree dispatch only runs `if (!consumed)` — all RectNode/Node mouse handlers silently stop working while app-level `mousePressed()` virtuals still fire (confusing!). With clickable nodes, either skip `enableMouseInput()`, or move orbit to `setOrbitButton(right)` / `setDragModifier(shift)`. Debug trick: `get_selected_node` after an MCP `mouse_click` — null selection on a known RectNode proves dispatch never ran.
