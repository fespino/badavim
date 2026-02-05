# Part 05: Providers (Calling the Model)

## Goal
See how the plugin runs model calls through CLI tools while keeping a uniform pipeline.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 04](PART-04-PROMPTS.md), [Next: Part 06](PART-06-GEOMETRY-AND-MARKS.md).

## Key Files
- `lua/99/providers.lua`
- `lua/99/request/init.lua`

## Mini Diagram
```
Request -> Provider -> vim.system -> TEMP_FILE -> on_complete
```

## Longer Trace: Request Execution
1. A `Request` is created in the op and bound to a `RequestContext`.
2. `Request:start` finalizes context and concatenates prompt text.
3. The provider builds a CLI command with `_build_command`.
4. `vim.system` runs the process and streams stdout and stderr.
5. On exit, the provider reads `TEMP_FILE` and triggers `on_complete`.
6. The op applies edits or displays errors based on status.

## Inline Callouts
- `lua/99/request/init.lua` `Request.new` chooses the provider and sets state.
- `lua/99/request/init.lua` `Request:start` is the orchestration step.
- `lua/99/providers.lua` `BaseProvider:make_request` standardizes execution.
- `lua/99/providers.lua` `_build_command` is implemented per provider.
- `lua/99/providers.lua` `_retrieve_response` reads the temp file.
- `lua/99/request/init.lua` `Request:cancel` kills the running process.

## Code Snapshot
File: `lua/99/providers.lua`
```lua
local proc = vim.system(command, { text = true, stdout = ..., stderr = ... },
  vim.schedule_wrap(function(obj)
    if obj.code ~= 0 then
      once_complete("failed", "process exit code: " .. obj.code)
    else
      local ok, res = self:_retrieve_response(request)
      if ok then
        once_complete("success", res)
      end
    end
  end)
)
```

## Practical Notes
- Providers are thin adapters; the heavy lifting lives in `BaseProvider`.
- Output is never taken from stdout. It is always read from the temp file.

## Performance Checklist
- Avoid per-line logging of large stdout streams unless debugging.
- Keep `_build_command` minimal and avoid heavy string formatting in hot paths.
- Read the temp file once per request and avoid additional parsing passes.
- Consider throttling UI updates if stdout is very chatty.

## Profiling Snippet
File: `lua/99/providers.lua`
```lua
local start = vim.uv.hrtime()
local ok, res = self:_retrieve_response(request)
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("provider:retrieve_response %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/providers.lua`.
2. Find `_build_command` for `OpenCodeProvider`.
3. Trace `_retrieve_response` and confirm it reads `request.context.tmp_file`.
