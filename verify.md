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

## Step 4: Drive the App (input injection — opt-in)

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

## Step 5: ImGui Widgets (if the app uses tcxImGui)

**Raw `mouse_click` does NOT reach ImGui widgets** (no sapp_event path). Use the
dedicated ImGui tools, which work by label via Test Engine hooks. The app must call
`imguiSetup()` before `mcp::registerDebuggerTools()`.

| Tool | Arguments | Description |
|------|-----------|-------------|
| `imgui_get_widgets` | `window` (opt) | List widgets with labels, types, positions |
| `imgui_click` | `label`, `window` (opt) | Click widget by label |
| `imgui_input` | `label`, `text`, `window` (opt) | Type into an input widget |
| `imgui_checkbox` | `label`, `value` (opt), `window` (opt) | Toggle/set checkbox |

Workflow: `imgui_get_widgets` first to discover exact labels, then drive by label.

## Step 6: App-State Assertions (custom tools — the strongest evidence)

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

## Step 7: Clean Up

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
