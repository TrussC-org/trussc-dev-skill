# TrussC Development Skill

AI coding skill for [TrussC](https://github.com/TrussC-org/TrussC) — a lightweight creative coding framework based on sokol, with an API similar to openFrameworks.

## What's Included

| File | Contents |
|------|----------|
| `SKILL.md` | Entry point — development loop, build workflow, essential patterns, common pitfalls |
| `architecture.md` | Design philosophy — structuring apps with the Node hierarchy and Mods |
| `app-lifecycle.md` | App class, window, input, math, threading, file utilities |
| `utils.md` | Data utilities bundled in core — JSON (`tc::Json`), XML (`tc::Xml`), text/file IO, `getDataPath` |
| `graphics.md` | Drawing primitives, Color, Image/Pixels/Texture/Fbo/Shader |
| `node-system.md` | Node/RectNode hierarchy, events, layout, UI design patterns |
| `verify.md` | Runtime verification — MCP screenshots, input injection, ImGui driving, evidence rules |
| `addon-authoring.md` | Creating & publishing addons (registry, addon.json, CI) |

## Install

### Claude Code (claude-code)

```bash
# Clone into your skills directory
git clone https://github.com/TrussC-org/trussc-dev-skill.git ~/.claude/skills/trussc-dev
```

### Cursor (2.4+)

Cursor supports Agent Skills natively and also auto-discovers skills from the
Claude/Codex directories — the Claude Code install above works as-is. For a
project-local install, clone into `.cursor/skills/` (or `.agents/skills/`)
inside your repository instead. Invoke manually by typing `/` in Agent chat.

### Codex

```bash
git clone https://github.com/TrussC-org/trussc-dev-skill.git ~/.codex/skills/trussc-dev
```

## Usage

The skill activates automatically when writing TrussC C++ code. It provides the AI with knowledge of:

- Build system (CMake presets, trusscli)
- API patterns (Node hierarchy, event-driven rendering, method chaining)
- Thread safety rules (Pixels vs Texture/Image/Fbo)
- UI widget design patterns (ScrollContainer, LayoutMod, Event/EventListener)
- 15 common pitfalls and how to avoid them

## License

MIT
