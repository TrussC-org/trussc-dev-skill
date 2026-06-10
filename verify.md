# Verifying TrussC Apps (Build → Run → Drive → Evidence → Verdict)

TrussC apps have built-in MCP automation: every app can be launched with an HTTP
control endpoint that provides screenshots, input injection, ImGui widget driving,
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
trusscli build                  # or: trusscli run (build + launch)
```

After adding/deleting/renaming source files, run `trusscli update` first
(Xcode does not pick up new GLOB files automatically; Ninja does).

## Step 2: Launch with MCP

```bash
TRUSSC_MCP=1 TRUSSC_MCP_PORT=8080 ./bin/myApp.app/Contents/MacOS/myApp &
sleep 2
```

- Fixed port (8080 here) is easiest for scripting. With no port set, the OS assigns
  one and prints it to stderr: `[MCP] HTTP server listening on http://localhost:PORT/mcp`
- All requests are JSON-RPC 2.0 over `POST http://localhost:PORT/mcp`.
- Send `initialize` once before calling tools.
- This works headlessly enough for CI-ish verification on macOS — no screen-recording
  (TCC) permission needed, because the app screenshots itself.

```bash
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{}}'
```

## Step 3: Screenshot (always available)

```bash
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":2,"params":{"name":"save_screenshot","arguments":{"path":"/tmp/verify_01.png"}}}'
```

Then **Read /tmp/verify_01.png and look at it.**

- **Never use macOS `screencapture`** — Metal/GL apps capture black/wrong via OS tools.
- `get_screenshot` returns Base64 PNG inline; `save_screenshot` writes a file (preferred for shell workflows).
- Event-driven apps (`setFps(EVENT_DRIVEN)` / independent fps): the screenshot shows the
  *last rendered* frame. Inject an input or wait for an update that calls `redraw()` first.
- Gotcha: if the app itself PNG-encodes every frame (e.g. frame recording), the MCP
  screenshot encoder can contend and produce white captures — pause recording while verifying.

## Step 4: Node Tree — Read and Write the Scene Graph

The most powerful handle of all: **the entire node tree is readable and writable over
MCP**. Positions, rotations, scales, colors, visibility — as numbers, not pixels. You
can inspect the exact state of every node and change it live, without rebuilding.

| Tool | Needs | Description |
|------|-------|-------------|
| `get_node_tree` | MCP only (always on) | Dump the tree as JSON: per node `{type, name, id, members, mods, children}` |
| `get_selected_node` | MCP only | The currently selected node (same selection an inspector shows) |
| `set_node_members` | debugger opt-in | Write reflected members of a node by id |
| `select_node` | debugger opt-in | Select a node by id (0 clears) — drives inspector selection |

("debugger opt-in" = the two `setup()` lines shown in Step 5.)

These tools share state with **tcxNodeInspector** (bundled addon): the inspector is
the human-side UI over the same reflection and the same selection — `select_node`
highlights the node in its Hierarchy panel, and a node clicked in the inspector is
what `get_selected_node` returns. Agent (MCP) and human (inspector) can debug the
same session side by side.

```bash
# Whole tree (use depth to keep output small; drill in via id)
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"get_node_tree","arguments":{"depth":1}}}'
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
  many were omitted — drill in with `get_node_tree {"id": <that node's id>}`.
- `mods` lists attached Mods as `{type, members}` — a Mod's `TC_REFLECT`ed members
  (e.g. LayoutMod spacing/padding, TweenMod params) are included.
- Give nodes names (`setName("scoreLabel")`) — trees with names are vastly easier to
  navigate than type-only dumps.

```bash
# Move, rotate, and scale node id=1 — applied immediately, next frame shows it
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":4,"params":{"name":"set_node_members","arguments":{"id":1,"members":{"pos":[420,420,0],"rotation":[0,0,15],"scale":[1.3,1.3,1.3]}}}}'
```

The response reports `applied` / `skipped` / `unknown` keys plus the node's resulting
members — you get read-back confirmation for free.

### The live-tuning loop (no rebuilds)

This is the core workflow these tools enable — layout and look tuning with **zero
builds in the loop**:

```
launch (TRUSSC_MCP=1)
  → get_screenshot           # see current state
  → get_node_tree            # read positions/rotations/colors as numbers
  → set_node_members         # nudge pos/color/rotation directly
  → get_screenshot           # check the look
  → repeat until it's right  # seconds per iteration, no compile
  → bake the final values into the C++ source   # the only build
```

Changes are runtime-only (lost on restart) — that's by design: tune live, then bake.
After baking, rebuild and take one confirming screenshot.

### Exposing your own members: TC_REFLECT

Base `Node` reflects `pos`, `globalPos`, `rotation`, `scale`, `visible`, `active`.
Any member your own Node subclass exposes via a `TC_REFLECT` block joins them —
readable in `get_node_tree`, writable via `set_node_members`, and editable in
tcxNodeInspector:

```cpp
class EnemyShip : public Node {
public:
    TC_REFLECT(EnemyShip)
        TC_FIELD(speed)                                  // plain member
        TC_PROPERTY(hullColor, getHullColor, setHullColor) // via getter/setter
    TC_REFLECT_END
    ...
};
```

Reflect your tuning parameters and the live-tuning loop covers game/app variables,
not just transforms. The same works inside Mods (`using Super = Mod;` + a
`TC_REFLECT` block) — built-in LayoutMod/TweenMod already expose their parameters.

## Step 5: Drive the App (input injection — opt-in)

Input injection requires two lines in the app's `setup()`:

```cpp
void tcApp::setup() {
    mcp::enableDebugger();          // opt-in: allows input injection
    mcp::registerDebuggerTools();   // registers mouse_*/key_* tools
}
```

These are inert unless `TRUSSC_MCP=1`, so it's fine to keep them in during development.
If they're missing and you need to drive the app, add them (and mention it in your report).

| Tool | Arguments | Notes |
|------|-----------|-------|
| `mouse_move` | `x`, `y`, `button` | button held = drag |
| `mouse_click` | `x`, `y`, `button` | 0 = left, 1 = right |
| `mouse_scroll` | `dx`, `dy` | |
| `key_press` / `key_release` | `key` | sokol_app keycode — letters are UPPERCASE (`'A'` = 65) |

```bash
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"mouse_click","arguments":{"x":100,"y":200}}}'
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

**Raw `mouse_click` does NOT reach ImGui widgets** (no sapp_event path) — always use
these tools for ImGui UIs.

Requirements: tcxImGui addon + `imguiSetup()` in `setup()`. In MCP mode the tools
register automatically — look for `[MCP] ImGui tools registered` on stderr.

### Discover: imgui_get_widgets

```bash
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":3,"params":{"name":"imgui_get_widgets","arguments":{}}}'
```

Returns every widget rendered this frame:

```json
{"widgets":[
  {"label":"Slider","window":"Settings","type":"input","rect":{"x":18,"y":58,"w":165,"h":19}},
  {"label":"Click me!","window":"Settings","type":"button","rect":{...}},
  {"label":"Show Demo","window":"Settings","type":"checkbox","checked":false,"rect":{...}}
]}
```

- `type` is one of `button` / `checkbox` / `input` / `tree`. Checkboxes include
  `checked`, tree nodes include `opened`.
- **Sliders and drag fields classify as `input`** — they accept direct numeric entry.
- Compound widgets decompose: a `ColorEdit3("Background")` appears as `##X`, `##Y`,
  `##Z` input fields (per channel).
- Current text/numeric **values are NOT in the list** — read them from a screenshot
  (the values render as text), or expose a `get_state` custom tool.
- Only widgets actually rendered this frame appear — collapsed windows/sections hide
  theirs. `imgui_click` the tree/header label to open it first.

### Act: imgui_click / imgui_input / imgui_checkbox

| Tool | Arguments | Use for |
|------|-----------|---------|
| `imgui_click` | `label`, `window` (opt) | Buttons, tree headers, tabs, selectables |
| `imgui_input` | `label`, `text`, `window` (opt) | **Set a value**: replaces text in input widgets; enters numeric values into slider/drag widgets |
| `imgui_checkbox` | `label`, `value` (opt), `window` (opt) | Set checkbox state (idempotent with `value`) |

`imgui_input` semantics are *replace and commit* (select-all → type → Enter). On
slider/drag widgets it activates ImGui's Ctrl+Click temp input under the hood, so the
typed number lands exactly:

```bash
# Set a SliderFloat to exactly 0.85
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":4,"params":{"name":"imgui_input","arguments":{"label":"Slider","text":"0.85"}}}'

# Set one channel of ColorEdit3("Background") to 200
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":5,"params":{"name":"imgui_input","arguments":{"label":"##X","text":"200"}}}'
```

If a label exists in multiple windows the tool returns an "ambiguous" error — pass
`window`. Duplicate labels *within* one window collide for ImGui itself (first match
wins); give debug widgets unique labels.

> Requires TrussC dev ≥ 2026-06-10 (commit `ffccb52e`). Older builds had two bugs:
> select-all didn't fire on macOS (text was *inserted* mid-string instead of replaced)
> and slider/drag numeric entry didn't work at all (the click just jumped the slider).

### The debug loop this enables

Expose tuning parameters as ImGui widgets (sliders, checkboxes) during development —
then verification becomes a parameter sweep without recompiling:

1. `imgui_get_widgets` → find `Exposure` slider
2. `imgui_input` `Exposure` = `0.5` → screenshot → Read
3. `imgui_input` `Exposure` = `2.5` → screenshot → Read → compare
4. Assert the visual change matches the criterion; report with both screenshots

This also means: when building a TrussC app, **adding an ImGui debug panel is adding
an automation API for free** — every slider you add is a value an agent can set
precisely and a state it can observe.

## Step 7: App-State Assertions (custom tools — the strongest evidence)

For logic-heavy apps, screenshots are weak evidence. Expose state directly:

```cpp
// In setup() — makes internal state queryable during verification
mcp::tool("get_state", "Return current game state as JSON")
    .bind([this]() { return json{{"score", score}, {"phase", (int)phase}}; });

mcp::resource("app://board", "Current board")
    .mime("application/json")
    .bind([this]() { return board.toJSON(); });
```

Adding a small read-only `get_state` tool for verification purposes is encouraged —
it turns "the screenshot looks right" into "the state IS right". Remove or keep as
you and the user prefer; they only exist in MCP mode.

## Step 8: Clean Up

```bash
curl -s -X POST http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":9,"params":{"name":"quit","arguments":{}}}'
# fallback if unresponsive:
kill %1 2>/dev/null
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
| Clicking Start begins the game | imgui_click "Start" + get_state | PASS |
| Score renders top-right | save_screenshot → Read | PASS |
| Handles 0-item edge case | not run | UNVERIFIED |

Notes: <caveats, screenshots paths, anything the user should look at>
```
