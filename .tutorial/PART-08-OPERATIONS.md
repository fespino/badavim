# Part 08: Operations (User Actions)

## Goal
Understand the three main user actions and how each applies edits.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 07](PART-07-TREESITTER.md), [Next: Part 09](PART-09-UI-WINDOWS.md), [Build-Along: Visual Range](BUILD-ALONG-VISUAL-RANGE.md).

## Key Files
- `lua/99/ops/fill-in-function.lua`
- `lua/99/ops/over-range.lua`
- `lua/99/ops/implement-fn.lua`

## Mini Diagram
```
operation -> prompt -> request -> provider -> apply edit
```

## Longer Trace: fill_in_function
1. The op calls `editor.treesitter.containing_function` with the cursor position.
2. It sets `context.range` to the function range and marks the function body location.
3. It builds the prompt with `prompts.fill_in_function()` and optional rules.
4. `Request:start` runs the model and streams status lines to a mark.
5. On success, `update_file_with_changes` replaces the function text.

## Longer Trace: over_range (visual)
1. Visual marks are finalized with `set_selection_marks` in `init.lua`.
2. `Range.from_visual_selection` builds a range from `'<` and `'>`.
3. The op creates top and bottom marks for status display.
4. The model response replaces the range via `Range.from_marks`.

## Longer Trace: implement_fn
1. The op calls `ts.fn_call` to locate a function call node.
2. It places an insertion mark above the containing function.
3. On success, it inserts the generated function text at the mark.

## Inline Callouts
- `lua/99/ops/fill-in-function.lua` `update_file_with_changes` performs the replacement.
- `lua/99/ops/over-range.lua` `Range.from_marks` rebuilds the range after edits.
- `lua/99/ops/implement-fn.lua` `Mark.mark_above_func` chooses the insertion point.
- `lua/99/ops/request_status.lua` `RequestStatus.new` drives inline spinner output.

## Code Snapshot
File: `lua/99/ops/over-range.lua`
```lua
local full_prompt = context._99.prompts.prompts.visual_selection(range)
request:add_prompt_content(full_prompt)
request:start({
  on_complete = function(status, response)
    if status == "success" then
      local new_range = Range.from_marks(top_mark, bottom_mark)
      new_range:replace_text(vim.split(response, "\n"))
    end
  end,
})
```

## Practical Notes
- The ops layer handles marks and buffer replacement, not the provider.
- Each op has a similar request lifecycle but different context building.

## Performance Checklist
- Limit status updates for long-running requests to reduce UI churn.
- Avoid copying large text blocks more than once during prompt assembly.
- Clean up marks promptly to avoid extmark growth over time.
- Keep per-op logic small and delegate shared work to helpers.

## Profiling Snippet
File: `lua/99/ops/over-range.lua`
```lua
local start = vim.uv.hrtime()
local new_range = Range.from_marks(top_mark, bottom_mark)
new_range:replace_text(vim.split(response, "\n"))
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("ops:visual_replace %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/ops/fill-in-function.lua`.
2. Find where `context.range` is set.
3. Compare its completion handler with `over-range.lua`.
