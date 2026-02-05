# Part 14: Model and Action Routing

## Goal
Abstract the backend model selection so different actions can route to different providers and models, such as a Copilot-like action using Claude Opus and a review action using ChatGPT Codex.

## See Also
See also: [Overview](OVERVIEW.md), [Prev: Part 13](PART-13-PERFORMANCE.md), [Back to Part 01](PART-01-GENERAL-CONCEPTS.md).

## Key Files
- `lua/99/init.lua`
- `lua/99/request-context.lua`
- `lua/99/request/init.lua`
- `lua/99/providers.lua`

## Mini Diagram
```
action -> routing table -> provider + model -> Request -> CLI
```

## Current State
- The default model is stored on `_99_state.model`.
- The provider can be overridden at setup with `opts.provider`.
- `RequestContext` copies the model from `_99_state.model`.
- Each provider knows how to build its CLI command.

## Strategy: Per-Action Routing Table
Create a routing table keyed by action name (operation) that picks a provider and a model. Examples:
- `copilot` -> Claude Opus (provider: ClaudeCodeProvider)
- `review` -> ChatGPT Codex (provider: OpenCodeProvider or another CLI wrapper)

### Example Routing Table (Concept)
```lua
local routes = {
  copilot = { provider = Providers.ClaudeCodeProvider, model = "claude-opus-4" },
  review = { provider = Providers.OpenCodeProvider, model = "gpt-4.1-codex" },
}
```

## Where to Apply It
1. When building the context in `get_context` (in `lua/99/init.lua`), select the route for the current action.
2. Set `context.model` and `context._99.provider_override` (or pass provider directly into Request).
3. In `Request.new`, prefer `context._99.provider_override` or a per-context provider.

## Suggested Implementation Sketch
### Step 1: Add Routes to State
```lua
-- inside _99_State.new() defaults
routes = {},
```

### Step 2: Extend Setup
```lua
function _99.setup(opts)
  _99_state.routes = opts.routes or {}
end
```

### Step 3: Use Routes in get_context
```lua
local function get_context(operation_name)
  _99_state:refresh_rules()
  local trace_id = get_id()
  local context = RequestContext.from_current_buffer(_99_state, trace_id)
  context.operation = operation_name

  local route = _99_state.routes[operation_name]
  if route then
    context.model = route.model or context.model
    context._99.provider_override = route.provider or context._99.provider_override
  end

  return context
end
```

### Optional Helper: Programmatic Route API
You can also provide a convenience method so users don't have to pass a full routes table.

```lua
-- in lua/99/init.lua
function _99.route(action, provider, model)
  _99_state.routes[action] = { provider = provider, model = model }
  return _99
end
```

## Example Usage in Config
```lua
require("99").setup({
  routes = {
    copilot = {
      provider = require("99.providers").ClaudeCodeProvider,
      model = "claude-opus-4",
    },
    review = {
      provider = require("99.providers").OpenCodeProvider,
      model = "gpt-4.1-codex",
    },
  },
})
```

## Example Usage With Helper
```lua
local _99 = require("99")
_99.route("copilot", _99.Providers.ClaudeCodeProvider, "claude-opus-4")
_99.route("review", _99.Providers.OpenCodeProvider, "gpt-4.1-codex")
```

## Practical Notes
- You can keep the default model for all actions unless a route overrides it.
- Route names should match `context.operation`, such as `fill_in_function`, `over-range`, or your own custom actions.
- If a provider CLI is not installed, fail early with a clear error window.

## Quick Exercise
1. Add a `routes` table in your config and map `fill_in_function` to a different model.
2. Trigger `fill_in_function` and verify the model string in logs.
3. Add a new action (ex: `review`) and route it to a second provider.
