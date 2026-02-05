# Part 07: Tree-sitter Queries

## Goal
Learn how the plugin locates functions and calls using Tree-sitter queries.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 06](PART-06-GEOMETRY-AND-MARKS.md), [Next: Part 08](PART-08-OPERATIONS.md).

## Key Files
- `lua/99/editor/treesitter.lua`
- `queries/<lang>/99-function.scm`
- `queries/<lang>/99-fn-call.scm`
- `queries/<lang>/99-imports.scm`

## Mini Diagram
```
cursor -> Tree-sitter query -> capture -> Range
```

## Longer Trace: Finding the Current Function
1. `treesitter.containing_function` resolves the parser and root node.
2. The query `99-function` is loaded for the current filetype.
3. Captures are iterated and the smallest `@context.function` range containing the cursor is selected.
4. `Function.from_ts_node` extracts the function and body ranges.
5. The op uses `function_range` to replace the full function text.

## Inline Callouts
- `lua/99/editor/treesitter.lua` `tree_root` resolves a parser and root node.
- `lua/99/editor/treesitter.lua` `containing_function` selects the smallest function range.
- `lua/99/editor/treesitter.lua` `Function.from_ts_node` maps captures to ranges.
- `lua/99/editor/treesitter.lua` `fn_call` finds a call node for `implement_fn`.
- `queries/<lang>/99-function.scm` must capture `@context.function` and `@context.body`.

## Code Snapshot
File: `lua/99/editor/treesitter.lua`
```lua
for id, node, _ in query:iter_captures(root, buffer, 0, -1, { all = true }) do
  local range = Range:from_ts_node(node, buffer)
  local name = query.captures[id]
  if name == "context.function" and range:contains(cursor) then
    found_range = range
    found_node = node
  end
end
```

## Practical Notes
- Query names map directly to files under `queries/<lang>/`.
- If a query is missing, the op fails early with a log message.

## Performance Checklist
- Cache loaded queries per filetype to avoid repeated `query.get` calls.
- Avoid scanning the entire tree when a smaller range can be used.
- Prefer early exits once the smallest containing function is found.
- Keep query files minimal to reduce capture overhead.

## Profiling Snippet
File: `lua/99/editor/treesitter.lua`
```lua
local start = vim.uv.hrtime()
local func = M.containing_function(context, cursor)
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("ts:containing_function %.2fms", ms))
```

## Quick Exercise
1. Open `queries/lua/99-function.scm` and inspect its capture names.
2. Compare it to `queries/go/99-function.scm`.
3. Trace how `Function.from_ts_node` uses `@context.body` in `treesitter.lua`.
