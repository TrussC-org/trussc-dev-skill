---
name: trussc-dev
description: TrussC framework knowledge — API patterns, node system, graphics, build workflow, common pitfalls. Use when writing TrussC C++ code.
---

# TrussC Development Guide

Comprehensive reference for writing TrussC applications. See topic files for detailed API.

## Quick Reference Files

- [node-system.md](node-system.md) — Node, RectNode, ScrollContainer, events, timers, mods
- [graphics.md](graphics.md) — Drawing, colors, style, Image/Pixels/Texture/Fbo
- [app-lifecycle.md](app-lifecycle.md) — App setup, window, input, logging, threading, math

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

### Addons

Addons live in `addons/` next to the project. The framework ships ~12 bundled (tcxBox2d, tcxCurl, tcxGltf, tcxHap, tcxImGui, tcxLua, tcxLut, tcxObj, tcxOsc, tcxQuadWarp, tcxTls, tcxWebSocket). Community addons are discovered through the topic-based registry.

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

**Registry / Browser:**
- Live browser: https://trussc.org/addons/
- registry.json: `https://raw.githubusercontent.com/TrussC-org/trussc-addons/gh-pages/registry.json`

**addon.json fields** (all optional, but encourage authors to fill in): `description`, `author`, `license`, `category`, `keywords`, `platforms`, `screenshot`, `demo_url`, `trussc_version`, `dependencies`. Categories come from a fixed set (`3d`, `ai`, `algorithms`, `animation`, `bridges`, `computer-vision`, `game`, `graphics`, `gui`, `hardware`, `machine-learning`, `network`, `physics`, `sound`, `typography`, `utilities`, `video`, `web`, `misc`). Dependencies declare other addons with optional tag/branch/commit pinning — `trusscli addon clone` walks the graph recursively with y/n prompts.

**Authoring (publishing your own):**
TrussC wants more addons — encourage authoring without ceremony. Listing in the registry is **automatic** (no PR). Crawler conditions:

1. Repo name matches `^tcx[A-Z]` (e.g. `tcxMyAddon`)
2. GitHub topic `trussc-addon` is set; repo public, not archived
3. `addon.json` at repo root

Minimal layout (matches bundled addons like `tcxBox2d`):

```
tcxMyAddon/
├── src/                # Addon code; .h + .cpp together
│   └── tcxMyAddon.h    # Umbrella header — users `#include "tcxMyAddon.h"`
├── example-basic/      # Optional sample (addons.make + shared CMakeLists.txt)
├── addon.json          # Registry metadata
└── CMakeLists.txt      # Only when FetchContent / vendored libs need a build
```

Pure header/source addons need no CMakeLists.txt — the parent app picks up `src/` automatically. Public symbols live under `tcx::` (alias of `trussc::ext`). Full authoring guide: `docs/ADDONS.md` in the TrussC repo.

**Build as you go** (no ceremony, do these while authoring — they're part of writing the addon, not a publication checklist):
- `addon.json` filled: `description`, `author`, `license`, `category` (fixed list), `keywords`, `screenshot` if visual. `{}` is technically enough but won't be discoverable.
- A LICENSE file. Without it, others legally can't use the addon.
- A README — even a few lines: what it does, how to include it, one snippet.
- Run it on the **current** TrussC release. **Don't set `trussc_version`** unless the addon truly can't run on the latest — leaving the field out means "works on the latest", which is the intended default. Treat `trussc_version` as a niche escape hatch, not standard practice.
- `platforms` lists only platforms you actually tested.

**Before suggesting the `trussc-addon` topic — light readiness check, NOT a gate.**

Adding the topic surfaces the addon in `trusscli addon list --remote`, tab completion, and trussc.org/addons — a public commitment. So before recommending the user add it, confirm intent and basic readiness. **Don't block enthusiasm**: combine the items below into **at most 3–5 questions in one round**, and skip anything already obviously resolved from the conversation. The point is "did you think about this?", not paperwork. Default disposition: encourage publishing.

- **Intent:** general-purpose for others, or project-specific glue? Glue can stay a public repo *without* the topic — that's a fine outcome, no shame in it.
- **Duplicate:** is there already a bundled or registry addon doing the same thing? If yes, is this meaningfully different?
- **Sanity:** built and ran on the latest TrussC; sample present (unless the addon is trivially obvious — a tiny wrapper may not need one); README exists; no secrets / API keys committed.
- **PRs:** open to merging the occasional pull request if one shows up? Not "actively maintain forever" — just "won't ghost a drive-by fix."

Once settled, adding the topic is one click in GitHub repo settings. Don't preempt by adding the topic first and then chasing the checklist.

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

### Event-Driven Rendering (redraw)

```cpp
setIndependentFps(VSYNC, 0);  // update runs at vsync, draw only on redraw()
```

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
