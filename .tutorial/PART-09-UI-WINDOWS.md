# Part 09: UI Windows and Prompt Capture

## Goal
Understand how the plugin gathers user input and displays errors or logs.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 08](PART-08-OPERATIONS.md), [Next: Part 10](PART-10-SKILLS-RULES.md).

## Key File
- `lua/99/window/init.lua`

## Mini Diagram
```
floating window -> user input -> BufWriteCmd -> callback
```

## Longer Trace: Prompt Window Lifecycle
1. `Window.capture_input` opens a floating window and scratch buffer.
2. The buffer is configured with `buftype=acwrite` and `bufhidden=wipe`.
3. Autocommands enforce focus and capture `BufWriteCmd` for submission.
4. `highlight_rules_found` scans for `@rule` tokens and highlights them.
5. On submit, the buffer contents are concatenated and passed to the callback.
6. The window is closed and all popups are cleared.

## Inline Callouts
- `lua/99/window/init.lua` `create_floating_window` centralizes window creation.
- `lua/99/window/init.lua` `capture_input` handles prompt editing and submit.
- `lua/99/window/init.lua` `highlight_rules_found` updates rule highlighting.
- `lua/99/window/init.lua` `display_error` shows fatal errors.
- `lua/99/window/init.lua` `display_full_screen_message` renders log output.
- `lua/99/window/init.lua` `clear_active_popups` closes all windows.

## Code Snapshot
File: `lua/99/window/init.lua`
```lua
vim.api.nvim_create_autocmd("BufWriteCmd", {
  group = group,
  buffer = win.buf_id,
  callback = function()
    local lines = vim.api.nvim_buf_get_lines(win.buf_id, 0, -1, false)
    local result = table.concat(lines, "\n")
    M.clear_active_popups()
    opts.cb(true, result)
  end,
})
```

## Practical Notes
- Prompt capture is key for operations with extra instructions.
- Cancellation uses `q` and `WinClosed` callbacks to avoid stale windows.

## Performance Checklist
- Reuse windows when possible instead of recreating them every time.
- Throttle rule highlighting updates for slow terminals.
- Avoid full-buffer redraws unless content changed.
- Keep floating window sizes proportional to avoid excessive reflow.

## Profiling Snippet
File: `lua/99/window/init.lua`
```lua
local start = vim.uv.hrtime()
local win = create_floating_window(config, { title = " 99 " })
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("ui:create_window %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/window/init.lua`.
2. List all autocommands registered inside `capture_input`.
3. Find where the prompt buffer filetype is set to `99`.
