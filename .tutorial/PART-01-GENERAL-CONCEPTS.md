# Part 01: General Concepts

## Goal
Build the mental model for how this Neovim plugin turns a user action into an LLM-driven edit.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Next: Part 02](PART-02-ENTRY-POINT-STATE.md), [Build-Along: Visual Range](BUILD-ALONG-VISUAL-RANGE.md).

## Key Files
- `README.md`
- `lua/99/init.lua`
- `lua/99/request/init.lua`
- `lua/99/request-context.lua`
- `lua/99/prompt-settings.lua`
- `lua/99/providers.lua`
- `lua/99/ops/fill-in-function.lua`

## Big Picture Flow
```
Keymap -> init.lua -> ops/* -> request -> provider -> TEMP_FILE -> buffer edit
```

## Longer Trace: fill_in_function End to End
1. The keymap calls `require("99").fill_in_function()` in `lua/99/init.lua`.
2. `get_context` builds a `RequestContext` with buffer metadata and a temp file path.
3. `ops.fill_in_function` uses Tree-sitter to find the containing function and sets `context.range`.
4. The op builds the prompt using `prompt-settings.lua` and adds it to the request.
5. `Request:start` concatenates prompt content and calls a provider.
6. `BaseProvider:make_request` runs a CLI via `vim.system` and reads `TEMP_FILE` on success.
7. The op replaces the function text using `Range:replace_text`.
8. Marks and virtual text are cleaned up via `make_clean_up`.

## Inline Callouts
- `lua/99/init.lua` `_99.fill_in_function` is the public entry point.
- `lua/99/ops/fill-in-function.lua` `fill_in_function` chooses the function range and handles completion.
- `lua/99/request-context.lua` `RequestContext:finalize` injects file context and temp file instructions.
- `lua/99/request/init.lua` `Request:start` is the request orchestration point.
- `lua/99/providers.lua` `BaseProvider:make_request` standardizes the CLI call.
- `lua/99/ops/marks.lua` `Mark:set_virtual_text` anchors status output.
- `lua/99/ops/request_status.lua` `RequestStatus:start` drives spinner updates.

## Code Snapshot
File: `lua/99/init.lua`
```lua
function _99.fill_in_function(opts)
  opts = process_opts(opts)
  ops.fill_in_function(get_context("fill_in_function"), opts)
end
```

## What to Internalize
- The model never edits the buffer directly. It only writes to a temp file.
- Tree-sitter determines what region is replaced, not the model.
- Every operation follows the same request pipeline.

## Responsive TUI Patterns (Like Responsive Web)
Neovim UIs can be responsive in the same spirit as web layouts. You can read the current UI size and adjust layout, visibility, or positioning to keep prompts and status views usable in small terminals.

### Example: Centered Window With Adaptive Sections
The snippet below keeps a window centered and shows or hides sections based on the terminal size.

```lua
local function get_ui_size()
  local ui = vim.api.nvim_list_uis()[1]
  return ui.width, ui.height
end

local function create_centered_config()
  local width, height = get_ui_size()
  local win_width = math.max(40, math.floor(width * 0.6))
  local win_height = math.max(6, math.floor(height * 0.3))
  return {
    width = win_width,
    height = win_height,
    row = math.floor((height - win_height) / 2),
    col = math.floor((width - win_width) / 2),
  }
end

local function render_prompt_lines()
  local width, height = get_ui_size()
  local lines = {}
  table.insert(lines, "99 Prompt")

  if width >= 100 then
    table.insert(lines, "Hints: use @rule to include skills")
  end

  if height >= 30 then
    table.insert(lines, "Context: selection + file")
  end

  return lines
end
```

Use this pattern in `lua/99/window/init.lua` if you want the prompt window to adapt to small terminals.

## Quick Exercise
1. Open any file with a simple function.
2. Run `:lua require("99").fill_in_function()`.
3. Open `lua/99/ops/fill-in-function.lua` and trace where the function range is chosen.
