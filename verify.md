# Verifying TrussC Apps (Build → Run → Drive → Evidence → Verdict)

TrussC native apps have built-in MCP automation: a dev build or explicit code grant
opens an HTTP endpoint for screenshots, input injection, ImGui widget driving,
and custom app-state tools. **Use this instead of guessing whether code works.**
"It compiles" is not verification.

## Verdict Vocabulary

Always report task completion with one of three verdicts:

| Verdict | Meaning |
|---------|---------|
| **COMPLETE** | All acceptance criteria verified (test run, screenshot inspected, or interaction driven) |
| **COMPLETE WITH NOTES** | Works, but with caveats the user should know (untested edge case, advisory warnings) |
| **BLOCKED** | Cannot verify, or verification failed. Say *why* and what you tried |

Rules:
- **NOT RUN ≠ FAIL**, but it also ≠ PASS. Unverified criteria must be listed explicitly.
- If **more than half** of the acceptance criteria are unverified, the verdict is BLOCKED, not COMPLETE.
- Never report COMPLETE based on a successful build alone.

## Evidence Matrix — what counts as verification

| Change type | Evidence required | Level |
|-------------|-------------------|-------|
| Logic / data / algorithm | Test run, or driven app interaction showing correct output | BLOCKING |
| Visual (drawing, layout, color) | MCP screenshot, actually viewed with the Read tool | BLOCKING |
| UI interaction (buttons, drag, scroll) | Driven walkthrough via MCP input/ImGui tools + screenshot | BLOCKING |
| App lifecycle (startup, quit, resize) | Launch, run a few seconds, screenshot, clean quit | BLOCKING |
| Config / asset path change | Smoke run (app launches, asset visible in screenshot) | ADVISORY |
| Comment / doc-only change | None (build still must pass) | — |

"Viewed with the Read tool" matters: saving a screenshot is not evidence until you
have actually looked at the PNG and confirmed it shows what the criterion demands.

## Step 1: Build

```bash
trusscli build --mcp            # v0.8 dev build for the verification steps below
```

After adding/deleting/renaming source files, run `trusscli update` first
(Xcode does not pick up new GLOB files automatically; Ninja does).

## Step 2: Launch with MCP

For development, `TRUSSC_MCP_PORT=8080 trusscli run --mcp` builds and launches.
Or launch the dev binary from Step 1 directly (macOS path shown):

```bash
TRUSSC_MCP_PORT=8080 ./bin/myApp.app/Contents/MacOS/myApp &
APP_PID=$!
sleep 2
```

`build --mcp` / `run --mcp` set `TRUSSC_MCP_DEV=ON`: Control access, automatic
server startup, `[MCP]` in the window title and a startup warning. A build without
`--mcp` explicitly turns it OFF again. Use ordinary builds for distribution.

Levels are **Public < ReadOnly < Control**. In an ordinary app, call
`mcp::setToolLevel(mcp::ToolLevel::ReadOnly)` in `setup()` for screenshots/status,
or `Control` for input and scene changes. This opens loopback HTTP; registration
of tools/resources alone never does. `mcp::startHttpServer()` explicitly opens
a listener for custom Public tools. List/call checks use the current grant;
the host grant survives hot reload, and code cannot lower a dev build's Control.

- `TRUSSC_MCP=1` or `TRUSSC_MCP_PORT` in a default Public build opens **only
  loopback `tc_get_health`**: no screenshots, status, alerts, custom tools or
  resources. With neither variable and no code opt-in, nothing listens.
- `TRUSSC_MCP=0` prevents the listener, including in dev builds.
- `TRUSSC_MCP_HOST` / `TRUSSC_MCP_TOKEN` are dev-only. Ordinary apps choose these
  with `mcp::startHttpServer(port, host, token)` **before** `setToolLevel()`; the
  first opener chooses the address, and mismatched later calls fail unchanged.
  Non-loopback binding requires a token.
- Fixed port (8080 here) is easiest for scripting. With no port set, the OS assigns
  one. The bound endpoint is logged at Notice level (desktop stdout, a configured
  log file, and `onLog`, subject to sink levels; `onLog` receives every line).
  Since v0.8 there is no raw copy on stderr: to discover a port, match
  `HTTP server listening on http://` in stdout or the `TRUSSC_LOG_FILE` file
  (the line carries the Logger's timestamp and `[NOTICE]` prefix), or skip
  parsing with a fixed `TRUSSC_MCP_PORT`. A console level of Warning or higher
  hides the line on stdout only.
- All requests are JSON-RPC 2.0 over `POST http://127.0.0.1:PORT/mcp`.
  MCP does not use stdout. Web builds have no HTTP listener.
- Send `initialize` once before calling tools.
- A working graphics/window session is still required; macOS screen-recording
  (TCC) permission is not, because the app captures its own framebuffer.

```bash
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{}}'
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":2,"params":{}}'
```

Check `tools/list` and `tc_get_health` (`toolLevel`, `buildKind`) for available
access. Log capture via `TRUSSC_LOG_FILE` and supervisor restarts need no MCP
grant; loopback health adds hang detection. Screenshots/status need ReadOnly.

## Step 3: Screenshot (ReadOnly or Control)

```bash
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":2,"params":{"name":"tc_save_screenshot","arguments":{"path":"/tmp/verify_01.png"}}}'
```

Wait for the file, then **Read /tmp/verify_01.png and look at it.**

- **Never use macOS `screencapture`** — Metal/GL apps capture black/wrong via OS tools.
- `tc_get_screenshot` returns an MCP image block (PNG by default; optional
  `format`, `width`, `quality`, `window`). `tc_save_screenshot` takes `path` and
  optional `window`; main-window saves queue a capture after present, so an `ok`
  reply means queued. It requests `redraw()` even in event-driven apps.
  `tc_get_screenshot` waits for a frame; use the save tool or arrange a redraw
  when the app's draw loop is idle.
- `window` is an index from `tc_list_windows` (default 0 = main). Secondary
  windows must be visible and render a frame; hidden windows fail immediately,
  and a capture waiting for a secondary frame times out after 5 seconds.

## Step 4: Node Tree — Read and Write the Scene Graph

The most powerful handle of all: **the entire node tree is readable and writable over
MCP**. Positions, rotations, scales, colors, visibility — as numbers, not pixels. You
can inspect the exact state of every node and change it live, without rebuilding.

| Tool | Needs | Description |
|------|-------|-------------|
| `tc_get_node_tree` | ReadOnly | Dump the tree as JSON; optional `id`, `depth`; per node `{type, name, id, members, derived, mods, children}` |
| `tc_get_selected_node` | ReadOnly | No args; the currently selected node (same selection an inspector shows) |
| `tc_set_node_members` | Control | Write `members` of a node by `id`; optional `mod` targets an attached Mod by short type name |
| `tc_select_node` | Control | Select a node by `id` (0 clears) — drives inspector selection |

(Control includes ReadOnly; use a dev build or the grant in Step 5.)

These tools share state with **tcxNodeInspector** (bundled addon): the inspector is
the human-side UI over the same reflection and the same selection — `tc_select_node`
highlights the node in its Hierarchy panel, and a node clicked in the inspector is
what `tc_get_selected_node` returns. Agent (MCP) and human (inspector) can debug the
same session side by side.

```bash
# Whole tree (use depth to keep output small; drill in via id)
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"tc_get_node_tree","arguments":{"depth":1}}}'
```

Per-node JSON:

```json
{"type":"UIButton","id":1,
 "members":{"pos":[50,50,0],"globalPos":[50,50,0],"rotation":[0,0,0],
            "scale":[1,1,1],"visible":true,"active":true},
 "mods":[{"type":"LayoutMod","members":{"spacing":4.0,"padding":[5,5,5,5]}}],
 "childCount":10}
```

- **Encoding:** Vec3 = `[x,y,z]`, Color = `[r,g,b,a]` (0–1), **rotation in degrees**
  (euler XYZ — the one MCP-facing exception to TrussC's radian convention).
- `depth` limits recursion; where children are cut off, `childCount` tells you how
  many were omitted — drill in with `tc_get_node_tree {"id": <that node's id>}`.
- `mods` lists attached Mods as `{type, members}` — a Mod's `TC_REFLECT`ed members
  (e.g. LayoutMod spacing/padding, TweenMod params) are included.
- Give nodes names (`setName("scoreLabel")`) — trees with names are vastly easier to
  navigate than type-only dumps.

```bash
# Move, rotate, and scale node id=1 — applied immediately, next frame shows it
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":4,"params":{"name":"tc_set_node_members","arguments":{"id":1,"members":{"pos":[420,420,0],"rotation":[0,0,15],"scale":[1.3,1.3,1.3]}}}}'
```

The response reports `applied` / `skipped` / `readOnly` / `unknown` keys plus the node's resulting
members — you get read-back confirmation for free.

### The live-tuning loop (no rebuilds)

This is the core workflow these tools enable — layout and look tuning with **zero
builds in the loop**:

```
launch (trusscli run --mcp, or an app that grants Control)
  → tc_get_screenshot           # see current state
  → tc_get_node_tree            # read positions/rotations/colors as numbers
  → tc_set_node_members         # nudge pos/color/rotation directly
  → tc_get_screenshot           # check the look
  → repeat until it's right    # seconds per iteration, no compile
  → bake the final values into the C++ source   # the only build
```

Changes are runtime-only (lost on restart) — that's by design: tune live, then bake.
After baking, rebuild with `--mcp` and take one confirming screenshot. Build
without `--mcp` for distribution; any explicit code grant still applies.

### Exposing your own members: TC_REFLECT

Base `Node` reflects `pos`, `globalPos`, `rotation`, `scale`, `visible`, `active`.
Any member your own class exposes via a `TC_REFLECT` block joins them — readable in
`tc_get_node_tree`, writable via `tc_set_node_members`, and editable in tcxNodeInspector.

One macro, `TC_VALUE`, covers everything — the arity decides the behavior:

```cpp
enum class WeaponType { Sword, Bow, Staff };
TC_ENUM_LABELS(WeaponType, "Sword", "Bow", "Staff")   // namespace scope, optional

struct Enemy : Node {
    float hp = 100;
    WeaponType weapon = WeaponType::Sword;
    bool isAlive() const { return hp > 0; }

    TC_REFLECT(Enemy, Node) {          // (Self, DirectBases...) — chains Node's block
        TC_VALUE(hp)                   // 1 arg: plain member, edited in place
        TC_VALUE(weapon)               // enums auto-detected (combo if TC_ENUM_LABELS)
        TC_VALUE(alive, isAlive)       // 2 args: getter only → read-only (greyed out)
        TC_VALUE(speed, getSpeed, setSpeed)  // 3 args: via setter (invariants run)
    }
};
```

- The braces are yours — no `TC_REFLECT_END`. Base classes are listed explicitly
  (`TC_REFLECT(Self, Base1, Base2...)`); a reflection root uses
  `TC_REFLECT_ROOT(Self) { ... }`.
- Supported types: float, int, bool, string, Vec2, Vec3, Color, and enums
  (`TC_ENUM_LABELS` gives them human-readable labels; without it they travel as int).

Reflect your tuning parameters and the live-tuning loop covers game/app variables,
not just transforms. The same works in Mods — `TC_REFLECT(MyMod, Mod) { ... }`;
built-in LayoutMod/TweenMod already expose their parameters.

## Step 5: Drive the App (input injection — Control)

Dev builds (`trusscli run --mcp`) already grant Control. In an ordinary build,
enable input injection with one line in the app's `setup()`:

```cpp
void tcApp::setup() {
    mcp::setToolLevel(mcp::ToolLevel::Control); // opens HTTP and grants control
}
```

This works without `TRUSSC_MCP=1`; `TRUSSC_MCP=0` prevents the listener. Prefer the
dev build for temporary verification. If adding a code grant, mention it in your report.

Compatibility only (deprecated, removal in v1.0.0): `registerInspectionTools()`
aliases `setToolLevel(ReadOnly)`; `registerControlTools()` and
`registerDebuggerTools()` alias `setToolLevel(Control)`. `enableDebugger()` is a
no-op. New code should use the grant API.

| Tool | Arguments | Notes |
|------|-----------|-------|
| `tc_mouse_move` | `x`, `y`; `button` (opt) | with `button` held, emits a drag; omit for a plain move |
| `tc_mouse_click` | `x`, `y`; `button`, `shift`, `ctrl`, `alt`, `super` (opt) | button defaults to 0 (left), 1 = right, 2 = middle. Press+release fire back-to-back |
| `tc_mouse_press` / `tc_mouse_release` | `x`, `y`; `button` (opt, default 0) | hold/release a button — bracket a drag gesture |
| `tc_mouse_scroll` | `dx`, `dy` | |
| `tc_key_press` / `tc_key_release` | `key`; `shift`, `ctrl`, `alt`, `super` (opt) | sokol_app keycode — letters are UPPERCASE (`'A'` = 65); pair press/release flags |

**Drag recipe** (EasyCam orbit, slider drags, node dragging):
`tc_mouse_press(x0,y0)` → `tc_mouse_move(x1,y1,button:0)` × N steps → `tc_mouse_release(xN,yN)`.
`tc_mouse_click` alone can NOT drag — its press+release fire in the same instant, so
drag consumers never see an open gesture.

```bash
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"tc_mouse_click","arguments":{"x":100,"y":200}}}'
```

Pattern: inject input → screenshot → Read → assert the expected change happened.

## Step 6: ImGui — Inspect and Drive the GUI (tcxImGui)

If the app uses tcxImGui, this is usually the **best debugging handle** — better than
raw mouse injection:

- You can *see inside* the GUI: every widget with label, type, window, and state.
- You target widgets **by label**, not pixel coordinates — robust against window
  position and layout changes.
- You can set values **directly and exactly**: type `0.85` into a slider instead of
  trying to hit the right pixel with a synthetic drag.

**Raw `tc_mouse_click` does NOT reach ImGui widgets** (no sapp_event path) — always use
these tools for ImGui UIs.

Requirements: tcxImGui addon + `imguiSetup()` in `setup()`. Tools register
regardless of grant; registration does not open HTTP. Widget/touched queries
require ReadOnly; interaction/reset requires Control. The registration message
`[MCP] ImGui tools registered (...)` uses `logNotice()` (desktop stdout).

### Discover: tcx_imgui_get_widgets

```bash
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"tcx_imgui_get_widgets","arguments":{}}}'
```

Returns widgets drawn in the last frame of each ImGui context. Optional `window`
filters by panel name; `windowId` filters by OS window (`tc_list_windows` index).
Example fields (rectangles and extra value metadata omitted here):

```json
{"widgets":[
  {"label":"Slider","window":"Settings","windowId":0,"type":"input","widget":"slider","valueType":"float","value":0.85,"touched":false},
  {"label":"Click me!","window":"Settings","windowId":0,"type":"button","touched":false},
  {"label":"Show Demo","window":"Settings","windowId":0,"type":"checkbox","checked":false,"value":false,"touched":false}
],"count":3}
```

- `type` is one of `button` / `checkbox` / `input` / `tree`. Checkboxes include
  `checked`, tree nodes include `opened`.
- **Sliders and drag fields classify as `input`** — they accept direct numeric entry.
- Value widgets report `widget`, `valueType`, `value`; text fields report their text
  except password fields.
  Composites such as `ColorEdit3("Background")` appear under their own label with
  an array value. Colors use 0–1 and report `colorSpace`; `SliderAngle` uses radians.
- Only widgets actually rendered in the last frame appear — collapsed windows/sections hide
  theirs. `tcx_imgui_click` the tree/header label to open it first.

### Act: tcx_imgui_click / tcx_imgui_input / tcx_imgui_checkbox

| Tool | Arguments | Use for |
|------|-----------|---------|
| `tcx_imgui_click` | `label`; `window`, `windowId` (opt) | Buttons, tree headers, tabs, selectables; composites cannot be clicked |
| `tcx_imgui_input` | `label`, `text`; `window`, `windowId` (opt) | **Set a value**: JSON encoded as a string for value widgets; replacement text for text fields |
| `tcx_imgui_checkbox` | `label`; `value`, `window`, `windowId` (opt) | Set checkbox state (idempotent with `value`; omit to toggle) |

`tcx_imgui_input` writes value widgets on their next frame and reads the value back,
then checks it again in the following frame. A changed widget returns true so app
change handlers run; there is no clamping to slider/drag min/max. Use the units and
array shape reported by `tcx_imgui_get_widgets`. Text fields use replace-and-commit
keystrokes. Inspect the reply for errors or app-side clamping/conversion:

```bash
# Set a SliderFloat to exactly 0.85
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":4,"params":{"name":"tcx_imgui_input","arguments":{"label":"Slider","text":"0.85"}}}'

# Set all three channels of ColorEdit3("Background") (RGB floats 0–1)
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":5,"params":{"name":"tcx_imgui_input","arguments":{"label":"Background","text":"[0.8,0.2,0.1]","window":"Settings"}}}'
```

If a label is ambiguous, pass `window` and, when needed, `windowId`. Give debug
widgets unique labels within each panel.

### The debug loop this enables

Expose tuning parameters as ImGui widgets (sliders, checkboxes) during development —
then verification becomes a parameter sweep without recompiling:

1. `tcx_imgui_get_widgets` → find `Exposure` slider
2. `tcx_imgui_input` `Exposure` = `0.5` → screenshot → Read
3. `tcx_imgui_input` `Exposure` = `2.5` → screenshot → Read → compare
4. Assert the visual change matches the criterion; report with both screenshots

For hand tuning, `tcx_imgui_get_touched` (ReadOnly, no args) reports edited values
(including inspector edits); bake them into source, then call
`tcx_imgui_reset_touched` (Control, no args). Value writes by `tcx_imgui_input` or
`tc_set_node_members` are not recorded there; text injection and clicks can be.

This also means: when building a TrussC app, **adding an ImGui debug panel is adding
an automation API for free** — every slider you add is a value an agent can set
precisely and a state it can observe.

## Step 7: App-State Assertions (custom tools — the strongest evidence)

For logic-heavy apps, screenshots are weak evidence. Expose state directly:

```cpp
// In setup() — makes internal state queryable during verification
mcp::tool("get_state", "Return current game state as JSON")
    .level(mcp::ToolLevel::ReadOnly)
    .bind([this]() { return json{{"score", score}, {"phase", (int)phase}}; });

mcp::resource("app://board", "Current board")
    .mime("application/json")
    .bind([this]() { return board.toJSON(); });
```

Adding a small read-only `get_state` tool for verification purposes is encouraged —
it turns "the screenshot looks right" into "the state IS right". Tools default to
Public when `.level()` is omitted; declare ReadOnly for inspection, Control for
mutation, or compile development-only tools out. Only `tc_` is reserved; custom
names such as `get_state` and the addon convention `tcx_<addon>_*` are allowed.
Registration never opens HTTP. Resources have no tool level: a full listener
exposes registered resources, while the environment-only health endpoint hides
them. Remove development-only resources if they should not be exposed.

## Step 8: Clean Up

```bash
curl -s -X POST http://127.0.0.1:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":9,"params":{"name":"tc_quit","arguments":{}}}'
# fallback if unresponsive, or when running with only ReadOnly access:
kill "$APP_PID" 2>/dev/null                 # PID saved by the direct launch in Step 2
```

Always quit/kill launched apps before finishing — leaked GUI processes hold the port
and confuse the next verification run.

## Fallback: GUI launch is flaky

macOS GUI launch can occasionally idle environment-wide (window appears but no frames /
MCP unresponsive). When that happens:

1. Retry once after killing the process.
2. Factor the logic under test into plain functions/classes that don't need a window,
   and exercise them from a tiny console `main` (GPU-free: `Pixels`, math, data types
   are all safe off-window).
3. If only visual verification remains impossible, report COMPLETE WITH NOTES or
   BLOCKED accordingly — never silently downgrade to "build passed".

## Report Template

```
Verdict: COMPLETE | COMPLETE WITH NOTES | BLOCKED

| Criterion | Method | Result |
|-----------|--------|--------|
| Clicking Start begins the game | tcx_imgui_click "Start" + get_state | PASS |
| Score renders top-right | tc_save_screenshot → Read | PASS |
| Handles 0-item edge case | not run | UNVERIFIED |

Notes: <caveats, screenshots paths, anything the user should look at>
```
