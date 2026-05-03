# AGENTS.md — Houdini Agent
## Overview
Houdini Agent is an AI-powered assistant that runs **inside SideFX Houdini** (not standalone). It has no standard Python packaging (no `setup.py`, `pyproject.toml`, `pip`). All dependencies are vendored in `lib/`. There is no CI/CD, no test suite, no linter/formatter config.
## Directory map
houdini_agent/          Main Python package (55 source files, ~38K LOC)
  main.py               Entry point (show_tool + module hot-reload)
  qt_compat.py          PySide2/PySide6 unified import layer
  core/                 Agent loop runner, main window, session manager
  ui/                   Qt widgets, theme, i18n, chat view, fonts
  skills/               9 pre-built Houdini analysis skills (auto-registered)
  utils/                AI client, MCP client, memory, plugin hooks, updater
houdini_agent_backup/   Mirror backup — NOT active code, never edit
lib/                    Vendored Python dependencies (requests, lxml, etc.)
config/                 Runtime config (gitignored except plugins.json, user_rules.json)
cache/                  Runtime cache (gitignored, created at runtime)
rules/                  User context rules (auto-loaded .md/.txt, _ prefix ignored)
plugins/                Community plugin extensions (_ prefix ignored)
shared/                 Shared utilities (config loading, history)
Doc/                    Offline Houdini docs (ZIP archives + knowledge base .txt)
trainData/              Training data exports (gitignored)
launcher.py             Bootstrap script — must run first, prepends lib/ to sys.path
VERSION                 Single-line version string (e.g. "1.5.5")
## Runtime environment
- **Runs only inside Houdini.** The app detects Houdini via `import hou`. If `hou` is not importable, it refuses to launch.
- **Houdini 20.5** ships PySide2; **Houdini 21+** ships PySide6. Always import Qt through `houdini_agent.qt_compat` — never import PySide2/PySide6 directly.
- **`lib/` takes priority** over system packages. `launcher.py` inserts `lib/` at `sys.path[0]` before anything else. Do not remove or reorder this.
- **Config** lives in `config/houdini_ai.ini` (gitignored, flat `key:value` format). Read via `shared/common_utils.py:load_config()`.
- **No standard toolchain.** There is no `pip install`, no virtualenv, no `pytest`, no `mypy`. The only way to run code is by loading it in Houdini.
## Development workflow
**Hot-reload is the development cycle.** When `main.show_tool()` is called, `_reload_modules()` force-reloads ~20 modules via `importlib.reload()`. To see changes:
1. Edit files in `houdini_agent/`
2. Close the Houdini Agent window
3. Re-launch (e.g. re-run the shelf tool)
4. The new module code is picked up via `_reload_modules()`
**Module reload quirks:**
- The reload list in `main.py:_reload_modules()` is **manually maintained**. If you add a new module that holds runtime state, add it to the reload list.
- Qt classes cannot be cleanly reloaded. If you change a UI class (widget, dialog), a full Houdini restart may be needed.
**Entry points:**
- `launcher.py:show_tool()` — bootstrap + auto-detect Houdini
- `houdini_agent/main.py:show_tool()` — reload modules + create/reuse window
- `houdini_agent/shelf_tool.py` — copy-paste code for Houdini Shelf Tool registration
## Architecture conventions
- **AITab is a Mixin composite.** The 7K-line `ui/ai_tab.py` inherits from 5 Mixin classes (`HeaderMixin`, `InputAreaMixin`, `ChatViewMixin`, `AgentRunnerMixin`, `SessionManagerMixin`). When editing, find the right mixin file first.
- **Thread safety.** All `hou` calls must run on the main thread. Use `qt_compat.invoke_on_main()` to dispatch from background threads. The MCP tool executor uses `BlockingQueuedConnection` for Houdini tools and plain threads for shell/web/doc.
- **Plugin system.** `utils/hooks.py` provides 7 lifecycle hooks. Plugins in `plugins/` are auto-discovered; files/folders starting with `_` are ignored. See `plugins/PLUGIN_DEV_GUIDE.md`.
- **User rules** live in `rules/` as `.md`/`.txt` files and are auto-injected into every AI request context. Files starting with `_` are ignored.
- **i18n.** All UI strings go through `ui/i18n.py` (~800 translations, Chinese/English). System prompts auto-adapt to the selected language.
## Version and release
- **Version** is stored in the root `VERSION` file (single line, e.g. `1.5.5`).
- **Release flow:** numbered branches `release/vX.Y.Z` created off `main`, merged back via PR.
- **Auto-updater** (`utils/updater.py`) checks GitHub Releases (owner: `Kazama-Suichiku`, repo: `Houdini-Agent`), downloads the release ZIP, and overwrites local files **except** `config/`, `cache/`, `trainData/`, `.git/`, `plugins/`, `rules/`. ETag caching in `cache/update_cache.json` avoids API rate limits.
## Git gotchas
- `config/houdini_ai.ini` is **gitignored** (contains API keys). Do not commit it.
- `trainData/` is gitignored (user-specific training exports).
- `*_backup.py` and `*_new.py` patterns are gitignored.
- The `houdini_agent_backup/` directory is not tracked in `.gitignore` but is a stale mirror — do not edit it, and do not add it as active code.
## Adding a new tool
1. Define the tool function in `utils/mcp/client.py` or in a skill file under `skills/`
2. Register it in `utils/tool_registry.py` with the appropriate mode/tags
3. If it's a skill, add `SKILL_INFO` dict and a `run(**kwargs)` function
4. For plugin tools, follow the `plugins/_example_plugin.py` pattern and the decorator `@tool`