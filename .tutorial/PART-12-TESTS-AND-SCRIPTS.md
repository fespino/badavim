# Part 12: Tests and Scripts

## Goal
Understand how the repo validates behavior and sets up Tree-sitter parsers for testing.

## See Also
See also: [Overview](OVERVIEW.md), [Walkthrough: Adding a Language](WALKTHROUGH.md), [Prev: Part 11](PART-11-LOGGING.md), [Next: Part 13](PART-13-PERFORMANCE.md), [Back to Part 01](PART-01-GENERAL-CONCEPTS.md).

## Key Files
- `lua/99/test/*`
- `scripts/tests/minimal.vim`
- `scripts/ci/install_treesitter_parsers.lua`

## Mini Diagram
```
minimal.vim -> load plugin -> run specs -> assert geometry/ops
```

## Longer Trace: Minimal Test Run
1. `scripts/tests/minimal.vim` configures runtime paths and Tree-sitter.
2. The minimal config checks for parsers and installs missing ones.
3. Specs run with the plugin on a controlled runtime path.
4. Geometry and mark behavior are validated first to avoid hidden off-by-one errors.
5. Operation specs validate visual replacement and provider behavior.

## Inline Callouts
- `scripts/tests/minimal.vim` sets `rtp` and initializes Tree-sitter.
- `scripts/ci/install_treesitter_parsers.lua` installs parsers in CI.
- `lua/99/test/geo_spec.lua` validates Point and Range behavior.
- `lua/99/test/marks_spec.lua` covers extmark creation and deletion.
- `lua/99/test/providers_spec.lua` checks provider behavior.
- `lua/99/test/visual_spec.lua` verifies visual range replacement.

## Code Snapshot
File: `lua/99/test/geo_spec.lua`
```lua
local Point = require("99.geo").Point
local Range = require("99.geo").Range

it("contains a point", function()
  local r = Range:new(0, Point:from_1_based(1, 1), Point:from_1_based(1, 5))
  assert.is_true(r:contains(Point:from_1_based(1, 3)))
end)
```

## Practical Notes
- Tests rely on Tree-sitter being available for the target languages.
- The minimal config attempts to install parsers if missing.

## Quick Exercise
1. Open `scripts/tests/minimal.vim` and see how it boots Neovim.
2. Read a spec file like `lua/99/test/visual_spec.lua`.
3. Identify which helper utilities are reused across specs.
