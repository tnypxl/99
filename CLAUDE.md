# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

99 is a Neovim plugin (Lua) that integrates with external AI CLI tools (opencode, claude, cursor-agent, kiro-cli) to provide AI-assisted code editing within Neovim. It restricts AI requests to specific file contexts and supports rule-based prompts via SKILL.md/AGENT.md files.

## Commands

### Build & CI

```bash
make pr_ready          # Run lint, test, and format check (the CI gate)
make lua_lint          # Lint with luacheck
make lua_test          # Run all tests with plenary.nvim
make lua_fmt           # Format with stylua
make lua_fmt_check     # Check formatting without modifying
```

### Running a Single Test

There is no built-in single-test target. Tests use plenary.nvim's busted runner and live in `lua/99/test/*_spec.lua`. To run tests you need Neovim, plenary.nvim, and nvim-treesitter available on the runtimepath (see `scripts/tests/minimal.vim` for paths).

## Important Conventions

- **This is not a standard Lua project.** Always use Neovim-provided functions (`vim.fn`, `vim.api`, `vim.uv`, `vim.fs`, `vim.system`, etc.) instead of Lua stdlib or external package resolution.
- **Lua dialect is LuaJIT** (Neovim standard). The `std` in `.luacheckrc` is `luajit`.
- **Formatting:** StyLua with 80-char column width, 2-space indentation, Unix line endings, double quotes preferred (`.stylua.toml`).
- **Type annotations:** The codebase uses LuaLS-style annotations (`@class`, `@param`, `@return`, `@field`, `@alias`) extensively. Follow this pattern for new code.

## Architecture

### Core Flow

1. User triggers an operation (e.g., `_99.visual()` from a visual-mode keymap)
2. `get_context()` builds a `RequestContext` from the current buffer, capturing file path, content, language, cursor/range, and auto-discovered AGENT.md files
3. A prompt capture window opens with nvim-cmp completions for `#rules` and `@files`
4. A `Request` object is created and executed via the configured **Provider**
5. The provider spawns an external CLI process (`vim.system`), streams output, reads the result from a temp file, and applies it back to the buffer

### Key Modules (`lua/99/`)

- **`init.lua`** — Plugin entry point, `_99_State` singleton, setup, public API (`visual`, `search`, `stop_all_requests`, `view_logs`)
- **`providers.lua`** — `BaseProvider` with `make_request`/`_retrieve_response`; implementations: `OpenCodeProvider`, `ClaudeCodeProvider`, `CursorAgentProvider`, `KiroProvider`
- **`request-context.lua`** — Immutable context for each request (buffer content, file path, language, model, temp file, auto-discovered md files)
- **`request/init.lua`** — Request lifecycle: process management, cancellation, state tracking
- **`ops/over-range.lua`** — The main visual-range replacement operation: sends context + prompt to AI, replaces the selected range with the response
- **`editor/treesitter.lua`** — Tree-sitter queries for extracting function signatures, bodies, and structural info per language
- **`language/`** — Per-language configurations (lua, go, java, elixir, cpp, ruby, typescript) defining tree-sitter query files and language-specific behavior
- **`extensions/agents/`** — Agent/skill system: discovers AGENT.md and SKILL.md files by walking up from the request file to project root; provides `#rule` completion
- **`extensions/files/`** — `@file` completion: discovers project files for fuzzy-search reference in prompts
- **`window/`** — Neovim floating window management for prompt input, log display, and status
- **`logger/`** — Request-scoped structured logging with multiple sinks (file, print, void)
- **`geo.lua`** — `Point` and `Range` primitives for buffer positions and visual selections

### Provider Pattern

All providers extend `BaseProvider` via metatables. Each provider implements `_build_command(query, request)` returning a CLI argument table and `_get_provider_name()`. The base class handles process spawning, stdout/stderr streaming, cancellation, and temp file result retrieval.

### Tree-sitter Queries

Language-specific `.scm` query files live in `queries/<language>/`. These are loaded by `editor/treesitter.lua` to extract function definitions, signatures, and bodies for context-aware AI requests. Required parsers for tests: `lua` and `typescript`.
