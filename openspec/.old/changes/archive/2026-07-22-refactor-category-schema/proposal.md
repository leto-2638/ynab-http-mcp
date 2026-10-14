## Why

The current `src/ynab_http_mcp/schemas/categories.py` is broken on import and inconsistent with the rest of the schema layer. `MCPCategoryGoal.from_ynab` has stray closing braces, an unreachable `output = "### Goal type: \n {}"` line outside any function, and an empty `explain_goal_funding_status` stub; `MCPCategory` has no `from_ynab` at all; `CategoriesResponse`/`CategoryResponse` use static `from_ynab_response` methods instead of the `MCPResponse.from_ynab` contract; and the file references undefined names (`StrictStr`, `StrictBool`, `StrictInt`, `Union`, `date`, `datetime`) plus a bogus `from unittest import case` import. The `accounts` schema and `tools/accounts.py` already show the target shape: a single core object (`MCPAccount`), a higher-level list wrapper (`MCPAccounts`) that holds a literal list of the core object and exposes a `HIDE_DELETED` switch, and a tools module that is two resources deep and calls `from_ynab` directly without wrappers or static helpers. We want the same shape for categories, plus LLM-friendly goal explanations, plus the redundant old types (`CategoriesResponse`, `CategoryResponse`, `CategoryGroup`) deleted outright rather than renamed.

## What Changes

- Delete the redundant old types outright: `CategoriesResponse`, `CategoryResponse`, and `CategoryGroup`. They are not renamed — they are removed, because the new design makes them unnecessary.
- Fix imports and type annotations in `schemas/categories.py` so the module imports cleanly under `mypy` and `ruff`.
- Complete `MCPCategoryGoal.from_ynab` so it produces `goal_summary` (what the goal is, in plain English) and `goal_status` (how the goal is progressing this period), using the existing `explain_goal_type` match-case and a finished `explain_goal_funding_status` helper.
- Add `MCPCategory.from_ynab` mapping `ynab.Category` (or wrapped `ynab.CategoryResponse`, unwrapped internally) → `MCPCategory`, including the nested `MCPCategoryGoal.from_ynab` call.
- Add `MCPCategories(MCPResponse[ynab.CategoriesResponse])` as the higher-level flat-list wrapper with a `HIDE_DELETED` class constant, mirroring `MCPAccounts`.
- Update `schemas/__init__.py` to export only `MCPCategory`, `MCPCategoryGoal`, `MCPCategories`.
- Rewrite `tools/categories.py` to mirror `tools/accounts.py`: two resources, `MCPCategory.from_ynab` for the single endpoint, `MCPCategories.from_ynab` for the list endpoint, `UUID(category_id)` for the path parameter, and no unused imports (`json`, `Literal`, `Annotated`, `MCPTransactions`).
- Add unit tests for `MCPCategory.from_ynab`, `MCPCategoryGoal.from_ynab` (each goal-type branch and the funding-status helper), and the `HIDE_DELETED` filter on `MCPCategories`.

**BREAKING**: The wire shape of both `data://categories` and `data://categories/{category_id}` changes. `data://categories` now returns a flat `{"categories": [...]}` list instead of `{"category_groups": [{"categories": [...]}]}`. `data://categories/{category_id}` now returns a single `MCPCategory` instead of `{category, category_group}`. The deleted `CategoriesResponse`, `CategoryResponse`, and `CategoryGroup` symbols are removed outright.

## Capabilities

### New Capabilities
- `category-resource-schema`: LLM-friendly translation of `ynab.Category` (and its wrapped `CategoriesResponse` / `CategoryResponse` containers) into MCP schemas that inherit from `MCPResponse[T]`, with a flat-list higher-level wrapper that filters deleted entries by default. No grouped wrapper.

### Modified Capabilities
- `ynab-mcp-resources`: the Categories Resource scenario's schema reference changes from `CategoriesResponse` to `MCPCategories`, and the wire contract now states that `data://categories` returns a flat list (not category groups) and `data://categories/{category_id}` returns a single `MCPCategory` (not `{category, category_group}`). Deleted categories are filtered out by default via `HIDE_DELETED`.

## Impact

- `src/ynab_http_mcp/schemas/categories.py` (rewrite to fix, complete, and drop `CategoriesResponse`/`CategoryResponse`/`CategoryGroup`)
- `src/ynab_http_mcp/schemas/__init__.py` (export only the three new names)
- `src/ynab_http_mcp/tools/categories.py` (rewrite to mirror `tools/accounts.py`)
- `openspec/specs/ynab-mcp-resources/spec.md` (delta for new wire shape)
- `tests/` (new unit tests for the three `from_ynab` entry points; existing tests that referenced the deleted symbols must be updated or removed)
- No new external dependencies; `pydantic` typing already imported via `base.py`.
