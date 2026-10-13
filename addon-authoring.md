# Authoring & Publishing TrussC Addons

Read this when *creating* an addon (not just using one). For installing/searching
addons in an app project, see SKILL.md § Addons.

## Layout

Minimal layout (matches bundled addons like `tcxBox2d`; full conventions live in
`addons/tcxTemplate` in the TrussC repo — read it when scaffolding):

```
tcxMyAddon/
├── src/                # Addon code; .h + .cpp together
│   └── tcxMyAddon.h    # Umbrella header — users `#include "tcxMyAddon.h"`
├── example-basic/      # Optional sample (addons.make + shared CMakeLists.txt)
├── addon.json          # Registry metadata
└── CMakeLists.txt      # Only when FetchContent / vendored libs need a build
```

- Pure header/source addons need no CMakeLists.txt — the parent app picks up `src/` automatically.
- Public symbols live under `tcx::` (alias of `trussc::ext`).
- Full authoring guide: `docs/ADDONS.md` in the TrussC repo.

## MCP tools (v0.8)

Register tools during addon setup regardless of the current grant. Only `tc_` is
reserved for core (registration is rejected); `tcx_` is free for apps and addons.
Conventionally use `tcx_<addon>_*` and declare a level on every tool:

```cpp
mcp::tool("tcx_example_reset", "Reset the addon state")
    .level(mcp::ToolLevel::Control)
    .bind(std::function<json()>([this] {
        reset();
        return json{{"status", "ok"}};
    }));
```

Use ReadOnly for inspection and Control for mutation. Omitting `.level()` means
Public; choose that deliberately or compile development-only tools out.
Registration never starts the listener. The app owns the grant: let it call
`mcp::setToolLevel(mcp::ToolLevel::ReadOnly)` or `Control` in `setup()`; do not
grant access from an addon. Control includes ReadOnly, and list/call checks use
the current grant regardless of registration order.

For development, use `trusscli run --mcp`; a build without `--mcp` clears the dev
switch. `TRUSSC_MCP=1` / `TRUSSC_MCP_PORT` in a default Public build exposes only
loopback `tc_get_health`, not addon tools. Check the startup exposure log or
`tools/list` for third-party tools' levels; see [verify.md](verify.md) for the
listener/environment rules and deprecated aliases.

## Registry (automatic listing — no PR)

TrussC wants more addons — encourage authoring without ceremony. The crawler lists a
repo automatically when:

1. Repo name matches `^tcx[A-Z]` (e.g. `tcxMyAddon`)
2. GitHub topic `trussc-addon` is set; repo public, not archived
3. `addon.json` at repo root

- Live browser: https://trussc.org/addons/
- registry.json: `https://raw.githubusercontent.com/TrussC-org/trussc-addons/gh-pages/registry.json`

**addon.json fields** (all optional, but encourage authors to fill in): `description`,
`author`, `license`, `category`, `keywords`, `platforms`, `screenshot`, `demo_url`,
`trussc_version`, `dependencies`. Categories come from a fixed set (`3d`, `ai`,
`algorithms`, `animation`, `bridges`, `computer-vision`, `game`, `graphics`, `gui`,
`hardware`, `machine-learning`, `network`, `physics`, `sound`, `typography`,
`utilities`, `video`, `web`, `misc`). Dependencies declare other addons with optional
tag/branch/commit pinning — `trusscli addon clone` walks the graph recursively with
y/n prompts.

## Build as you go

No ceremony — do these *while* authoring, they're part of writing the addon, not a
publication checklist:

- `addon.json` filled: `description`, `author`, `license`, `category` (fixed list),
  `keywords`, `screenshot` if visual. `{}` is technically enough but won't be discoverable.
- A LICENSE file. Without it, others legally can't use the addon.
- A README — even a few lines: what it does, how to include it, one snippet.
- Run it on the **current** TrussC release. **Don't set `trussc_version`** unless the
  addon truly can't run on the latest — leaving the field out means "works on the
  latest", which is the intended default. Treat `trussc_version` as a niche escape
  hatch, not standard practice.
- `platforms` lists only platforms you actually tested.

## Before suggesting the `trussc-addon` topic — light readiness check, NOT a gate

Adding the topic surfaces the addon in `trusscli addon list --remote`, tab completion,
and trussc.org/addons — a public commitment. So before recommending the user add it,
confirm intent and basic readiness. **Don't block enthusiasm**: combine the items below
into **at most 3–5 questions in one round**, and skip anything already obviously
resolved from the conversation. The point is "did you think about this?", not paperwork.
Default disposition: encourage publishing.

- **Intent:** general-purpose for others, or project-specific glue? Glue can stay a
  public repo *without* the topic — that's a fine outcome, no shame in it.
- **Duplicate:** is there already a bundled or registry addon doing the same thing?
  If yes, is this meaningfully different?
- **Sanity:** built and ran on the latest TrussC; sample present (unless the addon is
  trivially obvious — a tiny wrapper may not need one); README exists; no secrets /
  API keys committed.
- **PRs:** open to merging the occasional pull request if one shows up? Not "actively
  maintain forever" — just "won't ghost a drive-by fix."

Once settled, adding the topic is one click in GitHub repo settings. Don't preempt by
adding the topic first and then chasing the checklist.

## CI

Standalone addons can use the org-owned reusable workflow — one small `ci.yml`:

```yaml
jobs:
  build:
    uses: TrussC-org/ci-actions/.github/workflows/build-addon.yml@v1
```

Convention-driven: addon name = repo name, platforms from `addon.json`, builds every
`example-*/`, runs `tests/` if present. A `setup_script` hook exists for addons that
need external SDKs installed first.
