# Build-Along: Visual Range Replace (Minimal Plugin)

## Goal
Implement a minimal "visual replace" operation in your own plugin, mirroring the `over_range` workflow in this repo.

## See Also
See also: [Overview](OVERVIEW.md), [Part 03](PART-03-REQUEST-CONTEXT.md), [Part 04](PART-04-PROMPTS.md), [Part 05](PART-05-PROVIDERS.md), [Part 06](PART-06-GEOMETRY-AND-MARKS.md), [Part 08](PART-08-OPERATIONS.md).

## Step 1: Create a Minimal Module
Create a file like `lua/mini_assistant/init.lua` with a single public function.

```lua
local M = {}

function M.visual_replace()
  -- implemented below
end

return M
```

## Step 2: Capture the Visual Selection
Use the built-in marks `'<` and `'>` to get the range.

```lua
local function get_visual_range()
  local start_pos = vim.fn.getpos("'<")
  local end_pos = vim.fn.getpos("'>")
  local s_row, s_col = start_pos[2] - 1, start_pos[3] - 1
  local e_row, e_col = end_pos[2] - 1, end_pos[3] - 1
  return s_row, s_col, e_row, e_col
end
```

## Step 3: Build a Strict Prompt
You can copy the prompt contract idea from this repo. Keep it strict and explicit.

```lua
local function build_prompt(selection_text, tmp_file)
  return string.format([
[==[
You receive a selection in Neovim that you must replace with new code.
Return only code and write only to TEMP_FILE.
<SELECTION>
%s
</SELECTION>
<TEMP_FILE>%s</TEMP_FILE>
]==],
    selection_text,
    tmp_file
  )
end
```

## Step 4: Run the Model via CLI
This example uses `opencode`, but any CLI can work.

```lua
local function run_model(prompt, tmp_file, on_done)
  local cmd = { "opencode", "run", "-m", "opencode/claude-sonnet-4-5", prompt }
  vim.system(cmd, {}, function(obj)
    if obj.code ~= 0 then
      return on_done(false, "model failed")
    end
    local lines = vim.fn.readfile(tmp_file)
    return on_done(true, table.concat(lines, "\n"))
  end)
end
```

## Step 5: Replace the Range
Use `nvim_buf_set_text` to replace the selected range.

```lua
local function replace_range(buf, s_row, s_col, e_row, e_col, text)
  local lines = vim.split(text, "\n")
  vim.api.nvim_buf_set_text(buf, s_row, s_col, e_row, e_col, lines)
end
```

## Step 6: Wire It Together
Combine the steps into `visual_replace`.

```lua
function M.visual_replace()
  local buf = vim.api.nvim_get_current_buf()
  local s_row, s_col, e_row, e_col = get_visual_range()
  local selection = vim.api.nvim_buf_get_text(buf, s_row, s_col, e_row, e_col, {})
  local selection_text = table.concat(selection, "\n")
  local tmp_file = vim.fn.tempname()

  local prompt = build_prompt(selection_text, tmp_file)
  run_model(prompt, tmp_file, function(ok, result)
    if ok then
      replace_range(buf, s_row, s_col, e_row, e_col, result)
    else
      vim.notify(result, vim.log.levels.ERROR)
    end
  end)
end
```

## Step 7: Add a Keymap
In your Neovim config:

```lua
vim.keymap.set("v", "<leader>vr", function()
  require("mini_assistant").visual_replace()
end)
```

## Common Pitfalls
- Visual selection marks are 1-based; `nvim_buf_set_text` expects 0-based positions.
- Ensure the model only writes to the temp file, not to stdout.
- Do not mutate the buffer until the model call completes.

## Next Iterations
- Add progress status using extmarks, similar to `RequestStatus`.
- Add Tree-sitter context to the prompt for better results.
- Add `@rule` support like this repo's skills system.
