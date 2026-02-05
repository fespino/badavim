# Part 13: Performance Optimizations

## Goal
Apply performance thinking inspired by Casey Muratori's articles and videos, including his refterm work and the "antagonizing Microsoft shell" talk: measure first, reduce work, and keep the hot path simple.

## See Also
See also: [Overview](OVERVIEW.md), [Prev: Part 12](PART-12-TESTS-AND-SCRIPTS.md), [Next: Part 14](PART-14-MODEL-ABSTRACTION.md), [Back to Part 01](PART-01-GENERAL-CONCEPTS.md).

## Key Files
- `lua/99/init.lua`
- `lua/99/request-context.lua`
- `lua/99/providers.lua`
- `lua/99/window/init.lua`
- `lua/99/extensions/agents/helpers.lua`
- `lua/99/logger/logger.lua`

## Mini Diagram
```
user action -> gather context -> build prompt -> run CLI -> apply edits -> UI updates
```

## Longer Trace: Where Time and Allocations Accumulate
1. Every request calls `_99_state:refresh_rules`, which scans rule directories.
2. `RequestContext:finalize` reads markdown files and appends large context blocks.
3. Prompt assembly does multiple `table.insert` and `table.concat` operations.
4. The provider runs a CLI and then reads the entire temp file.
5. UI status updates run on a timer and update virtual text frequently.
6. Logging writes and fsyncs on each log statement.

## Inline Callouts
- `lua/99/init.lua` `_99_state:refresh_rules` does file scanning per request.
- `lua/99/request-context.lua` `_read_md_files` reads files from disk on every request.
- `lua/99/providers.lua` `_retrieve_response` reads the entire temp file at once.
- `lua/99/window/init.lua` `display_full_screen_message` recreates windows.
- `lua/99/ops/request_status.lua` `RequestStatus:start` updates at a fixed interval.
- `lua/99/logger/logger.lua` `FileSink:write_line` fsyncs on each log write.

## Principles to Apply (Muratori-Inspired)
- Measure first and track where time and allocations are spent.
- Avoid doing the same work repeatedly on the hot path.
- Reduce allocations in frequently called code.
- Batch or throttle UI updates.
- Prefer stable data structures and cached results.

## Profiling Hooks With `vim.uv.hrtime()`
You can add simple timing probes around hot paths to measure real cost in milliseconds. Use `vim.uv.hrtime()` for high-resolution timing.

## Profiling Conventions
To make performance data easy to compare across parts, keep a consistent naming and formatting scheme.

### Labeling Guidelines
- Use a `module:action` pattern, for example `context:finalize` or `rules:refresh`.
- Keep labels stable even if implementation details change.
- Prefer action names that map to a single function call site.

### Output Format
- Log a single JSON-style line with `label` and `ms` when possible.
- Use `logger:debug` during development and `vim.notify` for ad-hoc runs.

### Suggested Labels
- `rules:refresh`
- `context:finalize`
- `prompt:visual_selection`
- `provider:retrieve_response`
- `ui:create_window`
- `ts:containing_function`

### Minimal Timing Helper
```lua
local function time_block(label, fn, logger)
  local start = vim.uv.hrtime()
  local ok, result = pcall(fn)
  local elapsed_ms = (vim.uv.hrtime() - start) / 1e6
  if logger then
    logger:debug("perf", "label", label, "ms", elapsed_ms)
  else
    vim.notify(string.format("%s: %.2fms", label, elapsed_ms))
  end
  if not ok then
    error(result)
  end
  return result
end
```

### Example: Timing Rule Refresh
```lua
time_block("refresh_rules", function()
  _99_state:refresh_rules()
end, Logger:set_id(0))
```

### Example: Timing Context Finalize
```lua
time_block("context_finalize", function()
  context:finalize()
end, context.logger)
```

## Where to Place Timers
- Around `_99_state:refresh_rules` in `lua/99/init.lua`.
- Around `RequestContext:finalize` in `lua/99/request-context.lua`.
- Around `BaseProvider:make_request` completion in `lua/99/providers.lua`.
- Around window creation in `lua/99/window/init.lua`.

## Concrete Optimization Ideas for This Repo
- Cache rules and refresh only on explicit user action or on a debounce timer.
- Cache `md_files` contents by path and mtime to avoid re-reading unchanged files.
- Add size limits to full-file context to avoid large prompt assembly in big files.
- Reduce `RequestStatus` update frequency for slow terminals or low power systems.
- Reuse windows instead of closing and recreating them for repeated prompts.
- Add a logger mode that buffers and fsyncs less frequently when performance matters.

## Code Snapshot
File: `lua/99/logger/logger.lua`
```lua
function FileSink:write_line(str)
  local success, err = vim.uv.fs_write(self.fd, str .. "\n")
  if not success then
    error("unable to write to file sink", err)
  end
  vim.uv.fs_fsync(self.fd)
end
```

## Quick Exercise
1. Open `lua/99/init.lua` and find the comment above `_99_State:refresh_rules` about performance.
2. Identify a place you could cache results without changing behavior.
3. Write a short plan for how you would validate the performance impact.
