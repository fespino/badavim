# Part 11: Logging and Debugging

## Goal
Understand how logs are structured, stored, and surfaced.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 10](PART-10-SKILLS-RULES.md), [Next: Part 12](PART-12-TESTS-AND-SCRIPTS.md).

## Key Files
- `lua/99/logger/logger.lua`
- `lua/99/logger/level.lua`
- `lua/99/init.lua`

## Mini Diagram
```
request -> logger -> sink -> cache -> view_logs()
```

## Longer Trace: From Request to Log Viewer
1. `RequestContext.from_current_buffer` sets a logger with a request ID.
2. Each op logs with `logger:debug`, `logger:info`, or `logger:error`.
3. `Logger:_log` serializes logs as JSON and writes to the configured sink.
4. Logs are cached per request ID in `logger_cache`.
5. `Logger.logs()` returns a list of log arrays for display.
6. `require("99").view_logs()` shows the latest logs in a floating window.

## Inline Callouts
- `lua/99/logger/logger.lua` `Logger:configure` selects sinks and levels.
- `lua/99/logger/logger.lua` `Logger:file_sink` uses `vim.uv.fs_open`.
- `lua/99/logger/logger.lua` `Logger:_log` is the central logging method.
- `lua/99/logger/logger.lua` `Logger.logs` returns cached log lines.
- `lua/99/logger/logger.lua` `Logger:set_area` and `Logger:set_id` enrich metadata.
- `lua/99/init.lua` `view_logs` renders logs via `Window.display_full_screen_message`.

## Code Snapshot
File: `lua/99/logger/logger.lua`
```lua
function Logger:configure(opts)
  if not opts then
    return
  end
  if opts.level then
    self:set_level(opts.level)
  end
  if opts.type == "print" then
    self:print_sink()
  end
end
```

## Practical Notes
- Log statements require an ID or they assert.
- `print_on_error` will echo error logs to the user.

## Quick Exercise
1. Set the logger to file mode in your config.
2. Trigger a request and inspect the file output.
3. Run `:lua require("99").view_logs()` to verify log display.
