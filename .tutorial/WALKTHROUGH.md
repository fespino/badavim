# Walkthrough: Adding a New Language (Queries + Config)

This repo discovers functions and ranges through Tree-sitter queries. To add a new language, you'll wire in:

1. A language module in `lua/99/language/`.
2. Tree-sitter query files in `queries/<lang>/`.
3. The language name in `_99_state.languages` (defaults) or your config.

Below is a guided walkthrough you can follow step-by-step.

## Step 1: Choose the Neovim filetype name

The plugin indexes languages by Neovim `filetype`. For example:

- `typescript`
- `lua`
- `ruby`

Pick the exact filetype string you want to support, e.g. `python`.

## Step 2: Create the language module

Create `lua/99/language/python.lua` (or your filetype name). Minimal example:

```lua
local M = {}

--- @param item_name string
--- @return string
function M.log_item(item_name)
  return item_name
end

M.names = {
  body = "body",
}

return M
```

Notes:
- This file can remain minimal until you add language-specific behavior.
- The current system mostly uses Tree-sitter queries rather than this file.

## Step 3: Add Tree-sitter query files

Create a folder for your language:

- `queries/python/`

Then add at least:

- `queries/python/99-function.scm`

This file must capture:

- `@context.function`
- `@context.body`

Example layout (you must use node names that match your language's Tree-sitter grammar):

```
(function_definition) @context.function

(function_definition
  body: (block) @context.body)
```

Optional (but recommended) queries:

- `queries/python/99-fn-call.scm`
- `queries/python/99-imports.scm`

These support `implement_fn` and any future import-aware prompt context.

## Step 4: Add the language to defaults

In `lua/99/init.lua`, update the default languages list in `_99_State.new()`:

```lua
languages = { "lua", "go", "java", "elixir", "cpp", "ruby", "python" },
```

If you don't want to change defaults, you can set `_99.setup({ languages = { ... }})` if you add that option in your own fork. In this repo, defaults are hard-coded.

## Step 5: Validate with a simple test

Open a file in the new language and try:

- `:lua require("99").fill_in_function()`
- Visual selection + `:lua require("99").visual()`

If Tree-sitter queries are correct, the function range should be detected and replaced.

## Step 6: Iterate on the queries

If the plugin can't find functions or body nodes:

- Inspect your Tree-sitter grammar.
- Adjust query node names.
- Re-run the operation.

Start by checking existing language query folders for patterns. For example:

- `queries/lua/99-function.scm`
- `queries/go/99-function.scm`

## Step 7: Optional tests

If you want parity with existing tests, create a new spec in `lua/99/test/` that covers:

- Function detection in your language.
- Visual range replacement.

This is optional but strongly recommended to keep regressions low.
