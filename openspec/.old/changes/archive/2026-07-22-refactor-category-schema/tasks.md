## 1. Schema Cleanup

- [x] 1.1 Remove broken imports from `schemas/categories.py` (drop `from unittest import case`; remove unused `Self`, `StrictStr`, `StrictBool`, `StrictInt`, `Union`, `date`, `datetime` references)
- [x] 1.2 Delete `CategoriesResponse`, `CategoryResponse`, and `CategoryGroup` outright — they are not renamed or replaced
- [x] 1.3 Add the right typed imports (`Optional`, `List`, `Self`, `StrictStr`, `StrictBool`, `StrictInt`, `Union`, `date`, `datetime`, `UUID`) and re-bind them to the canonical aliases from `schemas/base.py` (`uuid_type`, `date_type`, `datetime_type`)
- [x] 1.4 Confirm `uv run mypy src` and `uv run ruff check src/ynab_http_mcp/schemas/categories.py` succeed on the cleaned file before any logic is added back

## 2. MCPCategoryGoal.from_ynab

- [x] 2.1 Delete the stray closing brace and the unreachable `output = "### Goal type: \n {}"` line outside the inner function
- [x] 2.2 Implement module-level `_explain_goal_type(raw: ynab.Category) -> str` that covers TB, TBD, MF (cadence 0/1/2/3-12/13/14 with `goal_needs_whole_amount` branching), NEED, and an unknown fallback
- [x] 2.3 Implement module-level `_explain_goal_funding_status(raw: ynab.Category) -> str` that reads `goal_under_funded`, `goal_overall_funded`, `goal_overall_left`, `goal_percentage_complete`, `goal_months_to_budget`, and `goal_snoozed_at` to produce a one-line progress sentence
- [x] 2.4 Re-implement `MCPCategoryGoal.from_ynab` to return `cls()` when `raw.goal_type` is None, otherwise build the model with raw fields populated and `goal_summary` / `goal_status` filled from the two helpers
- [x] 2.5 Add `goal_summary: Optional[str]` and `goal_status: Optional[str]` fields to `MCPCategoryGoal` with descriptions matching the spec

## 3. MCPCategory.from_ynab

- [x] 3.1 Add `from_ynab(raw: ynab.Category | ynab.CategoryResponse) -> MCPCategory` that unwraps `raw.data.category` when given a `CategoryResponse`
- [x] 3.2 Map every required YNAB field to its `MCPCategory` counterpart (id, category_group_id, category_group_name, name, hidden, internal, deleted, note, formatted budget/activity/balance, formatted goal-under-funded, formatted goal-overall-funded, formatted goal-overall-left)
- [x] 3.3 Nest `MCPCategoryGoal.from_ynab(raw)` into `MCPCategory.goal`; leave `goal = None` when the raw has no goal

## 4. MCPCategories (flat-list wrapper)

- [x] 4.1 Add `class MCPCategories(MCPResponse[ynab.CategoriesResponse])` with `HIDE_DELETED = True` class constant and `categories: List[MCPCategory]`
- [x] 4.2 Implement `MCPCategories.from_ynab(raw: ynab.CategoriesResponse) -> MCPCategories` that iterates `raw.data.category_groups`, builds each `MCPCategory` via `MCPCategory.from_ynab`, skips entries where `HIDE_DELETED and category.deleted`, and returns the assembled list

## 5. Wire-up

- [x] 5.1 Update `schemas/__init__.py` to export only `MCPCategory`, `MCPCategoryGoal`, `MCPCategories`; remove the old `CategoriesResponse` / `CategoryResponse` / `CategoryGroup` exports
- [x] 5.2 Rewrite `tools/categories.py` to mirror `tools/accounts.py`: import only `MCPCategory`, `MCPCategories`, and `UUID`; both resource handlers call `.from_ynab` directly on the raw response; the single-category handler coerces `category_id` with `UUID(...)`; drop `json`, `Literal`, `Annotated`, and `MCPTransactions` imports
- [x] 5.3 Update or remove any test that imports `CategoriesResponse`, `CategoryResponse`, or `CategoryGroup`
- [x] 5.4 Run `uv run mypy src` and `uv run ruff check .` and resolve any remaining type or lint errors

## 6. Tests

- [x] 6.1 Add unit tests for `MCPCategoryGoal.from_ynab` covering each branch: no goal, TB, TBD, MF cadence 0/1/2/3-12/13/14, NEED, unknown goal type, and the `goal_cadence_frequency` null errors
- [x] 6.2 Add unit tests for `_explain_goal_funding_status` for funded / under-funded / over-funded / snoozed / no-target combinations
- [x] 6.3 Add unit tests for `MCPCategory.from_ynab` with both a bare `ynab.Category` and a wrapped `ynab.CategoryResponse`, asserting formatted fields and nested goal
- [x] 6.4 Add unit tests for `MCPCategories.from_ynab` default behaviour (deleted filtered) and for a subclass with `HIDE_DELETED = False`
- [x] 6.5 Add a unit test asserting that `schemas.categories` no longer exports `CategoriesResponse`, `CategoryResponse`, or `CategoryGroup`

## 7. Verification

- [x] 7.1 Run the full test suite and confirm new tests pass with no regressions
- [x] 7.2 Restart the MCP service via the `ynab-http-mcp-control` skill and exercise `data://categories` and `data://categories/{category_id}` to confirm both endpoints return valid JSON in the new flat / bare shapes
