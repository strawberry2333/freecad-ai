# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A FreeCAD workbench (addon, installed by linking the repo into FreeCAD's `Mod/` dir) that puts an LLM chat panel inside FreeCAD. The LLM either writes Python that gets executed (Plan/Act code path) or calls structured FreeCAD tools (Act tool-calling path). The same tool set is also exposed as an MCP server for external clients (Claude Code, Claude Desktop).

Constraints from CONTRIBUTING.md:
- Python 3.11+, **stdlib only** at runtime (`urllib`, `json`, `ssl`, `threading`); no third-party packages. HTTP/SSE for LLM providers and MCP is hand-rolled.
- Qt imports go through `freecad_ai/ui/compat.py` (PySide6 first, PySide2 fallback), never `PySide6` directly.
- FreeCAD API calls belong in tool handlers / executor, not UI code.
- Commit prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `ui:`, `revert:` (scoped forms like `fix(ui):` are used in history). PRs target `master`.

## Commands

```bash
# Unit tests (no FreeCAD needed; integration tests are excluded by default via pyproject addopts)
pytest tests/unit/ -v

# Single file / single test
pytest tests/unit/test_config.py -v
pytest tests/unit/test_fallback.py::TestName::test_name -v

# Integration tests: run real FreeCAD headless via an AppImage found in ~/bin/FreeCAD*.AppImage
# (Linux only; skipped automatically if none is found)
pytest tests/integration/ -v -m integration

# Re-extract UI strings for translation (pylupdate5), then compile with lrelease / translations/compile_ts.py
cd translations && bash update_translations.sh
```

Some unit tests (UI, `InitGui`, chat widget loop) import Qt via `compat.py` and fail with `ModuleNotFoundError` when neither PySide6 nor PySide2 is installed in the test environment. `pip install PySide6` in the venv to run them. The project is developed on Linux. On Windows with a non-UTF-8 locale (e.g. GBK), tests that open files without an explicit encoding fail with `UnicodeDecodeError`, and a few path tests assume POSIX. Setting `PYTHONUTF8=1` fixes nearly all of the encoding failures.

`tests/conftest.py` sets `FREECAD_AI_CONFIG_DIR` to a temp dir **before** `freecad_ai` is imported, because config paths are resolved at import time; don't import `freecad_ai` in a way that bypasses it. Use the `tmp_config_dir` fixture to redirect config paths per test; the config singleton is reset after every test automatically. Note `core/conversation.py` holds its own import-time copy of `CONVERSATIONS_DIR`, so module-level path constants must be patched in each module that copied them.

There is no linter or build step configured.

## Architecture

**Entry points**
- `InitGui.py`: workbench registration and FreeCAD commands (chat panel, toolbar MCP Server toggle, preferences page).
- `mcp_server_http.py`: run inside a GUI FreeCAD; serves MCP over HTTP (Streamable HTTP at `/mcp`, legacy HTTP+SSE at `/sse`). Env: `MCP_HOST`, `MCP_PORT`, `MCP_ALLOWED_HOSTS`, `MCP_AUTH_TOKEN`.
- `mcp_server_entry.py`: headless STDIO MCP server (`FreeCAD -c mcp_server_entry.py`). Stdout is the protocol channel, so nothing else may print to it.

**Threading model (the main source of subtle bugs).** FreeCAD's document API must only be touched on the Qt main thread. LLM streaming runs in a `QThread` (`_LLMWorker` in `ui/chat_widget.py`); each tool call is marshalled to the main thread via a queued signal and the worker blocks until the result comes back (5-minute backstop). `tools/executor_utils.py` provides the same main-thread dispatch for other callers (the MCP GUI server in `mcp/gui_server.py`, skill optimization). Destroying a running `QThread` is fatal, so workers are held until they finish.

**Tools** (`tools/`)
- `registry.py`: `ToolParam` / `ToolDefinition` / `ToolResult` / `ToolRegistry`, which converts params to JSON Schema for both OpenAI-style and Anthropic-style APIs.
- `freecad_tools.py`: all built-in tools (a large file). Adding a tool = a `_handle_*` function + a `ToolDefinition` constant + an entry in `ALL_TOOLS` at the bottom.
- `setup.py:create_default_registry`: built-in tools + user extension tools (`extensions/user_tools.py`, from the config dir and optionally FreeCAD's Macro dir) + tools proxied from connected MCP servers (`mcp/manager.py`).
- `reranker.py`: optional keyword/LLM filtering to send only the top-N relevant tools per turn.

**LLM layer** (`llm/`)
- `providers.py`: `PROVIDERS` dict; adding a provider is one entry with `api_style` of `"openai"` or `"anthropic"`.
- `client.py`: `LLMClient` speaks both API styles (streaming SSE, tool calls, thinking, vision probe). Request bodies are pinned by golden-fixture tests (`tests/unit/fixtures/golden_*_bodies.json`, `test_golden_request_bodies.py`), so request-shape changes must update the fixtures deliberately. `create_client` resolves a connection profile + params.
- `fallback.py`: walks a chain of fallback profiles when the primary model fails.

**Config** (`config.py`): a single `AppConfig` in `<FreeCADAI dir>/config.json` (not FreeCAD's `user.cfg`), accessed via `get_config()` / `save_current_config()`, with listeners (`add_config_listener` / `notify_config_changed`) so the UI reacts to changes. The directory is resolved at import: `$FREECAD_AI_CONFIG_DIR`, then the version-scoped FreeCAD user config dir (`.../v1-1/FreeCADAI`), then the legacy `~/.config/FreeCAD/FreeCADAI`, with one-shot migration and a sweep of stale copies. Settings UI: `ui/settings_dialog.py` and the FreeCAD preferences page (`ui/prefs_page.py`) share the pages in `ui/settings_pages/`, and both write the same file. Connection profiles (`ProviderConfig`) carry per-profile model, limits, and thinking.

**Code execution** (`core/executor.py`): code from the LLM goes through static validation, then a subprocess **sandbox** run in a separate FreeCAD process (catches C++ console errors and invalid shapes without touching the live doc), then real execution inside an undo transaction. `core/dangerous_mode.py` and `tools/macro_runner.py` gate running macros. `core/backups.py` handles document backups.

**Conversation / prompt**: `core/system_prompt.py` builds the system prompt from document context (`core/context.py`), AGENTS.md (`extensions/agents_md.py`), and skills. `core/conversation.py` holds history, compaction, and save/load (session resume). `core/active_document.py` resolves the GUI-aligned active document; use it rather than `App.ActiveDocument` directly.

**MCP** (`mcp/`)
- `server.py`: `MCPServer` exposes a `ToolRegistry`. It is a **dual-era** server: it answers both the legacy `initialize` handshake and the stateless 2026-07-28 per-request `_meta` style from one endpoint (`protocol.py` holds the revision table).
- `transport.py`: STDIO and HTTP transports.
- `client.py` / `manager.py`: this addon acting as an MCP *client* to external servers (with era negotiation).
- `gui_server.py`: `ServerController` behind the toolbar toggle.

**Extensions**
- Skills: `skills/<name>/SKILL.md` (built-in) or the config dir. Optional `VALIDATION.md` (geometry test cases) and `handler.py` (deterministic `execute(args)`). Invoked via `/command` or autonomously. `skill_evaluator.py` / `skill_validator.py` / `tools/optimize_tools.py` implement `/optimize-skill`.
- Hooks: `hooks/registry.py`. Python files with events `pre_tool_use`, `post_tool_use`, `user_prompt_submit`, `post_response`, `file_attach`. Built-in examples are in top-level `hooks/`.

**i18n**: UI strings go through `freecad_ai/i18n.py` `translate(context, text)`; the German `.ts`/`.qm` files are in `translations/`.

## Design docs

Non-trivial features have a design spec and an implementation plan in `docs/superpowers/specs/` and `docs/superpowers/plans/` (older ones in `docs/specs/`), dated by filename. Check for an existing spec before changing a feature, and follow the same spec-then-plan pattern for new ones. `CHANGELOG.md` is updated at release time (`chore(release): vX.Y.Z-alpha`).
