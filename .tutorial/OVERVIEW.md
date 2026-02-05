# Overview: Building a Neovim LLM Assistant (This Repo)

This repo is an educational, Neovim-first implementation of an LLM-powered code assistant. The core idea is simple:

1. Capture context from the current buffer and selection.
2. Build a strict prompt contract.
3. Call an external LLM CLI (via `vim.system`).
4. Read model output from a temp file.
5. Apply edits in the buffer using Tree-sitter ranges and extmarks.
6. Provide UI feedback via floating windows and virtual text.

Everything is built using Neovim APIs (`vim.api`, `vim.uv`, `vim.treesitter`, `vim.system`) rather than Lua stdlib tooling. The code is designed to be hackable and visible rather than "abstracted away."

## Diagram: End-to-End Flow

```
User Action (keymap)
  |
  v
lua/99/init.lua
  - resolve rules
  - build RequestContext
  - choose operation
  |
  v
lua/99/ops/*
  - select range/function
  - place extmarks
  - build prompt
  |
  v
lua/99/request/init.lua
  - finalize context
  - send prompt
  |
  v
lua/99/providers.lua
  - build CLI command
  - vim.system(...)
  - read TEMP_FILE
  |
  v
lua/99/ops/*
  - replace text via Range / extmarks
  |
  v
UI feedback
  - virtual text status
  - floating windows
```

## Parts (Learning Roadmap)

1. **General Concepts**
   - Overall architecture and contract with the model.
   - Files: `README.md`, `lua/99/init.lua`

2. **Entry Point, State, and Configuration**
   - Setup, state tracking, request history.
   - Files: `lua/99/init.lua`, `lua/99/id.lua`, `lua/99/time.lua`

3. **Request Context (What gets sent)**
   - Buffer, filetype, rules, selection text.
   - Files: `lua/99/request-context.lua`, `lua/99/utils.lua`

4. **Prompt Design and Output Contract**
   - Strict output-only-to-temp-file model.
   - Files: `lua/99/prompt-settings.lua`

5. **Providers (Calling the model)**
   - Base provider, CLI adapters.
   - Files: `lua/99/providers.lua`, `lua/99/request/init.lua`

6. **Editing Geometry (Points, Ranges, Marks)**
   - Buffer replacements via Range + extmarks.
   - Files: `lua/99/geo.lua`, `lua/99/ops/marks.lua`, `lua/99/ops/request_status.lua`

7. **Tree-sitter Queries**
   - Locating functions/calls/imports.
   - Files: `lua/99/editor/treesitter.lua`, `queries/*/99-function.scm`, `queries/*/99-fn-call.scm`

8. **Operations (User Actions)**
   - Fill function, visual replace, implement function.
   - Files: `lua/99/ops/fill-in-function.lua`, `lua/99/ops/over-range.lua`, `lua/99/ops/implement-fn.lua`

9. **UI Windows and Prompt Capture**
   - Floating prompt input and error display.
   - Files: `lua/99/window/init.lua`

10. **Skills/Rules and Completion**
   - `SKILL.md` discovery and `@` completion.
   - Files: `lua/99/extensions/agents/*`, `lua/99/extensions/cmp.lua`

11. **Logging and Debugging**
   - Sinks, request logs, quickfix listing.
   - Files: `lua/99/logger/*`, `lua/99/init.lua`

12. **Tests and Scripts**
   - Minimal test harness and Tree-sitter setup.
   - Files: `lua/99/test/*`, `scripts/tests/minimal.vim`, `scripts/ci/install_treesitter_parsers.lua`

13. **Performance Optimizations**
   - Measuring, reducing allocations, and smoothing UI updates.
   - Files: `lua/99/request-context.lua`, `lua/99/providers.lua`, `lua/99/window/init.lua`, `lua/99/extensions/agents/helpers.lua`

14. **Model and Action Routing**
   - Abstracting providers and selecting models per action.
   - Files: `lua/99/init.lua`, `lua/99/request-context.lua`, `lua/99/request/init.lua`, `lua/99/providers.lua`

## How to Use This Tutorial

Start with `WALKTHROUGH.md` to learn how language support is added, then go part-by-part using the `PART-*.md` files for deeper dives into each area.
