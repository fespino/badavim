# Part 04: Prompt Design and Output Contract

## Goal
Understand how prompts enforce safe, deterministic edits.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 03](PART-03-REQUEST-CONTEXT.md), [Next: Part 05](PART-05-PROVIDERS.md).

## Key File
- `lua/99/prompt-settings.lua`

## Mini Diagram
```
operation prompt + context + TEMP_FILE rules -> final prompt string
```

## Longer Trace: Prompt Build for Visual Selection
1. `ops.over_range` calls `prompts.visual_selection(range)`.
2. That function captures the selection text and the full file contents.
3. If a user prompt is provided, it is wrapped with `prompts.prompt` into `<DIRECTIONS>`.
4. `RequestContext:finalize` appends `<TEMP_FILE>` and the MustObey rules.
5. `Request:start` concatenates everything into a single prompt string.

## Inline Callouts
- `lua/99/prompt-settings.lua` `prompts.output_file` forbids non-temp-file output.
- `lua/99/prompt-settings.lua` `prompts.read_tmp` tells the model not to read the temp file.
- `lua/99/prompt-settings.lua` `prompts.prompt` wraps user instructions in `<DIRECTIONS>`.
- `lua/99/prompt-settings.lua` `prompts.visual_selection` embeds selection and file context.
- `lua/99/prompt-settings.lua` `tmp_file_location` adds `<MustObey>` and `<TEMP_FILE>`.
- `lua/99/prompt-settings.lua` `get_file_location` and `get_range_text` encode file and range details.

## Code Snapshot
File: `lua/99/prompt-settings.lua`
```lua
tmp_file_location = function(tmp_file)
  return string.format(
    "<MustObey>\n%s\n%s\n</MustObey>\n<TEMP_FILE>%s</TEMP_FILE>",
    prompts.output_file(),
    prompts.read_tmp,
    tmp_file
  )
end
```

## Practical Notes
- `visual_selection` includes the entire file content, which can be large.
- `fill_in_function` insists on returning a full function signature.

## Ephemeral Reasoning Output (Stdout vs TEMP_FILE)
Some providers can emit intermediate text on stdout. In this repo, that output is treated as **ephemeral** and only used for status display, never for file edits. The only authoritative output is what the model writes to `TEMP_FILE`.\n
Key points:
- Stdout is streamed into virtual text via `RequestStatus` for user feedback.
- Edits are applied exclusively from the contents of `TEMP_FILE`.
- If a provider emits reasoning or partial code on stdout, it should **not** be applied.
- The prompt contract in `prompt-settings.lua` reinforces this by instructing the model to write only to `TEMP_FILE`.

If you build your own provider, make sure stdout is treated as transient UI feedback and **never** as a source of edits.

## Performance Checklist
- Cap full-file context for `visual_selection` to avoid huge prompts.
- Reuse template strings where possible instead of rebuilding each time.
- Prefer `table.concat` once at the end rather than repeated concatenations.
- Keep prompts minimal to reduce token load and latency.

## Profiling Snippet
File: `lua/99/ops/over-range.lua`
```lua
local start = vim.uv.hrtime()
local full_prompt = context._99.prompts.prompts.visual_selection(range)
local ms = (vim.uv.hrtime() - start) / 1e6
vim.notify(string.format("prompt:visual_selection %.2fms", ms))
```

## Quick Exercise
1. Open `lua/99/prompt-settings.lua`.
2. Compare `fill_in_function` and `visual_selection` prompt text.
3. Find exactly where `<TEMP_FILE>` is injected into the final prompt.
