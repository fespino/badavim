# Part 06: Editing Geometry (Points, Ranges, Marks)

## Goal
Understand how buffer locations are modeled and how edits remain stable during async requests.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 05](PART-05-PROVIDERS.md), [Next: Part 07](PART-07-TREESITTER.md).

## Key Files
- `lua/99/geo.lua`
- `lua/99/ops/marks.lua`
- `lua/99/ops/request_status.lua`

## Mini Diagram
```
Point + Point -> Range -> replace_text()
           Mark -> virtual text / insert text
```

## Longer Trace: Visual Selection to Replacement
1. `Range.from_visual_selection` reads `'<` and `'>` marks from Neovim.
2. The op creates extmarks above and below the range with `Mark.mark_above_range`.
3. During request execution, `RequestStatus:start` updates virtual lines at a mark.
4. On success, the op reconstructs a `Range` from marks and calls `replace_text`.
5. Cleanup removes extmarks and stops virtual text updates.

## Inline Callouts
- `lua/99/geo.lua` `Point:from_cursor`, `Point:to_vim`, and `Point:from_ts_point` handle index conversions.
- `lua/99/geo.lua` `Range.from_visual_selection` normalizes line-mode selections.
- `lua/99/geo.lua` `Range:replace_text` wraps `nvim_buf_set_text`.
- `lua/99/ops/marks.lua` `Mark:set_text_at_mark` inserts text at an extmark.
- `lua/99/ops/marks.lua` `Mark:set_virtual_text` renders status output lines.
- `lua/99/ops/request_status.lua` `RequestStatus:start` schedules spinner updates.

## Code Snapshot
File: `lua/99/geo.lua`
```lua
function Range:replace_text(replace_with)
  local s_row, s_col = self.start:to_vim()
  local e_row, e_col = self.end_:to_vim()
  vim.api.nvim_buf_set_text(self.buffer, s_row, s_col, e_row, e_col, replace_with)
end
```

## Practical Notes
- The project uses 1-based Points internally, but Neovim APIs are 0-based.
- Extmarks are essential for stability when buffers change mid-request.

## Performance Checklist
- Avoid creating many Point objects in tight loops; reuse when possible.
- Use extmarks sparingly and stop status updates as soon as possible.
- Keep conversion helpers simple and avoid extra allocations.
- Prefer range-based edits over per-line edits for large changes.

## Profiling Snippet
File: `lua/99/geo.lua`
```lua
local start = vim.uv.hrtime()
range:replace_text(lines)
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("range:replace_text %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/geo.lua` and trace `Point:to_vim` and `Point:from_ts_point`.
2. Open `lua/99/ops/marks.lua` and find `Mark:set_virtual_text`.
3. Note how `RequestStatus` updates the spinner over time.
