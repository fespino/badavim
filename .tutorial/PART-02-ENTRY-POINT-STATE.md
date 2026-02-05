# Part 02: Entry Point, State, and Configuration

## Goal
Understand how the plugin initializes, stores runtime state, and exposes its public API.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 01](PART-01-GENERAL-CONCEPTS.md), [Next: Part 03](PART-03-REQUEST-CONTEXT.md).

## Key Files
- `lua/99/init.lua`
- `lua/99/id.lua`
- `lua/99/time.lua`

## Mini Diagram
```
setup() -> _99_state defaults -> languages -> extensions -> ready
```

## Longer Trace: Startup to First Request
1. `require("99").setup(opts)` constructs a fresh `_99_state`.
2. `Logger:configure` applies log settings early so errors are visible.
3. The model is selected from `opts.model` or provider defaults.
4. Rules are refreshed via `Agents.rules` and extensions are initialized.
5. A VimLeavePre autocmd ensures running requests are cancelled on exit.
6. When a user runs an op, `get_context` builds a `RequestContext` using the state.

## Inline Callouts
- `lua/99/init.lua` `_99_State.new` defines defaults like `model`, `languages`, and `ai_stdout_rows`.
- `lua/99/init.lua` `_99.setup` wires state, logger, model selection, and extensions.
- `lua/99/init.lua` `_99_state:refresh_rules` loads skills and updates completion.
- `lua/99/init.lua` `_99_state:add_active_request` tracks cancellable requests.
- `lua/99/init.lua` `stop_all_requests` is the global safety switch.
- `lua/99/id.lua` generates monotonically increasing request IDs.
- `lua/99/time.lua` provides timestamps for request history.

## Code Snapshot
File: `lua/99/init.lua`
```lua
function _99.setup(opts)
  _99_state = _99_State.new()
  _99_state.provider_override = opts.provider
  _99_state.display_errors = opts.display_errors or false
  _99_state:refresh_rules()
  Languages.initialize(_99_state)
  Extensions.init(_99_state)
end
```

## Practical Notes
- State lives in `_99_state`, which is treated as a singleton.
- Request history is stored in `_99_state.__request_history` for quickfix and logs.

## Performance Checklist
- Avoid refreshing rules on every request; debounce `_99_state:refresh_rules` where possible.
- Keep language initialization out of hot paths once setup is complete.
- Cap request history size to reduce memory churn in long sessions.
- Skip expensive logger sinks unless actively debugging.

## Profiling Snippet
File: `lua/99/init.lua`
```lua
local start = vim.uv.hrtime()
_99_state:refresh_rules()
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("rules:refresh %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/init.lua` and find `_99_State.new()`.
2. List the default fields and their values.
3. Find where `_99_state.model` is set in `_99.setup`.
