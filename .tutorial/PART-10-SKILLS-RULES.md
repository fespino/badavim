# Part 10: Skills, Rules, and Completion

## Goal
Understand how `SKILL.md` files are discovered and injected using `@` tokens.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 09](PART-09-UI-WINDOWS.md), [Next: Part 11](PART-11-LOGGING.md).

## Key Files
- `lua/99/extensions/agents/init.lua`
- `lua/99/extensions/agents/helpers.lua`
- `lua/99/extensions/cmp.lua`
- `lua/99/extensions/init.lua`

## Mini Diagram
```
SKILL.md -> rule index -> @token -> injected context
```

## Longer Trace: From @token to Injected Context
1. The user opens a prompt window and types `@`.
2. CMP requests completion items from `CmpSource:complete`.
3. The user selects a rule name such as `@vim`.
4. `Agents.by_name` finds matching rules by token name.
5. `RequestContext:add_agent_rules` reads the file and wraps its content.
6. The wrapped rule is appended to `ai_context` before the request starts.

## Inline Callouts
- `lua/99/extensions/agents/helpers.lua` `ls` discovers `SKILL.md` files.
- `lua/99/extensions/agents/init.lua` `rules` builds `by_name` and `custom` lists.
- `lua/99/extensions/agents/init.lua` `by_name` resolves `@name` tokens.
- `lua/99/extensions/agents/init.lua` `find_rules` resolves `@path` tokens.
- `lua/99/request-context.lua` `add_agent_rules` injects rule contents.
- `lua/99/extensions/cmp.lua` `CmpSource:complete` supplies completion items.

## Code Snapshot
File: `lua/99/extensions/agents/init.lua`
```lua
for word in haystack:gmatch("@%S+") do
  local rule_string = word:sub(2)
  local rule = M.get_rule_by_path(rules, rule_string)
  if rule then
    table.insert(out, rule)
  end
end
```

## Practical Notes
- Completion uses CMP with a keyword pattern of `@\k+`.
- Rule docs shown in completion are the first lines of `SKILL.md`.

## Performance Checklist
- Cache discovered rules and refresh only when directories change.
- Avoid reading `SKILL.md` contents on every completion request.
- Limit docs preview length to keep completion snappy.
- Reuse the rules index between prompt captures.

## Profiling Snippet
File: `lua/99/extensions/agents/helpers.lua`
```lua
local start = vim.uv.hrtime()
local rules = M.ls(path)
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("rules:scan %.2fms", ms))
```

## Quick Exercise
1. Create a simple `SKILL.md` under a directory listed in `completion.custom_rules`.
2. Open the prompt window and type `@` to see completion.
3. Check logs to see the rule content appended to the AI context.
