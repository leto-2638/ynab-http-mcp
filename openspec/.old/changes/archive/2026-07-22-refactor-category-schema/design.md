## Context

`src/ynab_http_mcp/schemas/categories.py` is the read-side schema for YNAB categories. The current file is broken on import and inconsistent with the rest of the schema layer:

- `MCPCategoryGoal.from_ynab` contains a stray closing brace, an `output = "### Goal type: \n {}"` line that lives outside any function, and an empty `explain_goal_funding_status` helper.
- `MCPCategory` has no `from_ynab` classmethod.
- `CategoriesResponse` and `CategoryResponse` use static `from_ynab_response` methods, not the `MCPResponse.from_ynab` contract.
- The module imports `StrictStr`, `StrictBool`, `StrictInt`, `Union`, `date`, and `datetime` without binding them, and pulls in a bogus `from unittest import case`.
- `tools/categories.py` carries dead imports (`json`, `Literal`, `Annotated`, `MCPTransactions`) and wraps the two resources in two redundant response types that the `accounts` resource does not need.

The accounts schema (`schemas/accounts.py`) and tool module (`tools/accounts.py`) are the reference shape we are converging on: one core object (`MCPAccount(MCPResponse[ynab.Account])`), one higher-level list wrapper (`MCPAccounts(MCPResponse[ynab.AccountsResponse])`) that holds a flat `List[MCPAccount]` and exposes a `HIDE_DELETED` class constant, and a tool module that is just two resource definitions calling `from_ynab` directly. No `MCPAccountResponse` wrapper, no static helpers, no group aggregation. We want the same shape for categories, plus a human-readable `goal_summary` / `goal_status` pair so an LLM can summarise a YNAB goal without reading raw cadence integers.

## Goals / Non-Goals

**Goals:**
- Make `schemas/categories.py` import cleanly under `mypy` and `ruff`.
- Give every resource-side category model a working `from_ynab` that follows the `MCPResponse[T]` contract.
- Produce LLM-friendly `goal_summary` and `goal_status` strings on `MCPCategoryGoal`, covering all four goal types (`TB`, `TBD`, `MF`, `NEED`) and the cadence variants documented at <https://github.com/ynab/ynab-sdk-python/blob/main/docs/Category.md>.
- Provide a flat-list higher-level wrapper (`MCPCategories`) with a `HIDE_DELETED` class constant, mirroring `MCPAccounts`.
- Delete `CategoriesResponse`, `CategoryResponse`, and `CategoryGroup` outright — no renamed replacement.
- Rewrite `tools/categories.py` to mirror `tools/accounts.py` line-for-line in structure: two `@mcp.resource` handlers, no wrappers, no static helpers, no dead imports.
- Add unit tests covering `MCPCategory.from_ynab`, `MCPCategoryGoal.from_ynab` (each goal-type branch and the funding-status helper), and the `HIDE_DELETED` filter on `MCPCategories`.

**Non-Goals:**
- Adding write-side (`MCPRequest`) support — that is for a future change covering write tools.
- Refactoring other resource schemas (`payees`, `transactions`, `planning`, `budget_management`) — out of scope here; pattern can be replicated once this lands.
- Re-introducing any grouped response — the grouped form is removed permanently; callers that need it can derive groups client-side from `category_group_id`.
- Adding a `MCPTransactions` reference to `tools/categories.py` — that import was dead and goes away.

## Decisions

### Decision 1: All category resource models extend `MCPResponse[T]`

`MCPCategory(MCPResponse[ynab.Category])`, `MCPCategoryGoal(MCPResponse[ynab.Category])`, and `MCPCategories(MCPResponse[ynab.CategoriesResponse])` all inherit from `MCPResponse[T]`. This matches `MCPAccount`/`MCPAccounts` and the user's instruction to "inherit from the base" (corrected from `MCPRequest` to `MCPResponse` because resources are read-side). `MCPRequest` stays reserved for write tools.

**Alternatives considered:**
- Make `CategoryGroup` and `MCPRequest` share a base. Rejected — `CategoryGroup` is being deleted entirely, and `MCPRequest` is for write tools; we do not want a circular dependency between read and write paths.

### Decision 2: Delete old types outright instead of renaming

`CategoriesResponse`, `CategoryResponse`, and `CategoryGroup` are deleted. There is no `MCPCategoryWithGroup`, no `MCPGroupedCategories`, no renamed equivalents. The single-category endpoint calls `MCPCategory.from_ynab` directly, and `MCPCategories` is a flat list — neither needs a wrapper, neither needs groups.

**Alternatives considered:**
- Rename `CategoriesResponse` → `MCPCategories`, `CategoryResponse` → `MCPCategoryWithGroup`, `CategoryGroup` → `MCPCategoryGroup`. Rejected — `tools/accounts.py` has no equivalent rewrites (there is no `MCPAccountsResponse`, no `MCPAccountWithSomething`); carrying the wrappers forward for categories adds zero value and contradicts the requested simplification.

### Decision 3: Keep `MCPCategoryGoal` as a nested sub-model with both raw fields and explanation strings

`MCPCategoryGoal` keeps the full set of typed YNAB goal fields (so callers that want raw numbers keep access), and gains two new fields: `goal_summary: Optional[str]` (plain-English description of what the goal is) and `goal_status: Optional[str]` (how the goal is progressing in the current period). The `from_ynab` classmethod is the single source of truth that fills both — raw fields are populated directly, explanation strings are produced by `explain_goal_type` and `explain_goal_funding_status` helpers.

**Alternatives considered:**
- Collapse to flat strings only on `MCPCategory`. Rejected by user — raw typed access is still useful for callers like `MonthCategory` consumers.
- Hide `MCPCategoryGoal` behind a `MCPCategory.goal` optional slot. Rejected because it adds a layer without simplifying the JSON the LLM sees — flat nested keys are still readable.

### Decision 4: Centralise currency formatting via YNAB `_formatted` fields

`MCPCategory` exposes `budgeted_formatted`, `activity_formatted`, `balance_formatted`, and the goal-related formatted strings exactly as the YNAB SDK returns them. No custom milliunit → decimal conversion in this layer. This matches the prior decision recorded in `openspec/changes/archive/2026-07-21-add-budget-management-tools/proposal.md`.

**Alternatives considered:**
- Re-introduce a `currency_format.py` utility. Rejected — YNAB already formats correctly for the plan's currency and locale; re-formatting risks drift.

### Decision 5: Goal explanation helpers are private module functions, not class methods

`_explain_goal_type(raw: ynab.Category) -> str` and `_explain_goal_funding_status(raw: ynab.Category) -> str` are module-level (underscore-prefixed) helpers that take a raw `ynab.Category` and return a string. `MCPCategoryGoal.from_ynab` calls them once each and assigns to `goal_summary` / `goal_status`. This keeps the match-case branching out of the class body where the original broken code lived.

**Alternatives considered:**
- Static methods on `MCPCategoryGoal`. Equivalent — picked module functions to avoid re-binding `cls` and to keep the helpers reusable from `MCPCategory.from_ynab` if needed later.

### Decision 6: `tools/categories.py` mirrors `tools/accounts.py` line-for-line in structure

Two `@mcp.resource` handlers, no wrapper types, no static `from_ynab_response` helpers, no unused imports. The single-category handler coerces the path parameter with `UUID(category_id)` instead of the `Annotated[str, "..."]` form, matching accounts exactly.

**Alternatives considered:**
- Keep `Annotated[str, "ID of the category"]` because FastMCP uses it for tool metadata. Rejected — `tools/accounts.py` uses bare `str` and works fine; matching accounts is more important than metadata that no current caller reads.

## Risks / Trade-offs

- [Risk] `explain_goal_funding_status` has to cover every cadence / frequency combination YNAB allows (cadence 0/1/2/3-12/13/14) and could miss an edge case → Mitigation: unit tests for each goal_type branch (TB, TBD, MF with no cadence, MF monthly, MF weekly, MF every-x-months, MF yearly, MF every-2-years, NEED, unknown) plus a snapshot test that pins the exact phrasing of each summary.
- [Risk] Deleting `CategoriesResponse` / `CategoryResponse` / `CategoryGroup` breaks any external import → Mitigation: only `tools/categories.py` and the in-repo tests reference these symbols; both are updated in this change.
- [Risk] Changing the wire shape of `data://categories` and `data://categories/{category_id}` breaks MCP clients that expected the grouped form → Mitigation: this is the simplification the user explicitly requested; the new flat shape is identical to `data://accounts` and easier for LLMs to traverse.
- [Risk] Adding `goal_summary` / `goal_status` increases payload size per category → Mitigation: strings are short (one or two sentences) and only present when a goal exists, so the cost is negligible versus the LLM comprehension win.
- [Risk] YNAB may add new goal types in the future that we do not yet handle → Mitigation: the `Unknown goal type: {raw.goal_type}` branch in `_explain_goal_type` keeps the function total and gives the LLM a usable signal; a follow-up change can extend it.

## Migration Plan

1. Land `schemas/categories.py` rewrite. Old symbols are deleted in the same commit; no migration shim.
2. Update `schemas/__init__.py` and `tools/categories.py` to use the new names (and nothing else).
3. Run `uv run mypy src` and `uv run ruff check .` to confirm clean static analysis.
4. Run the new unit tests; update or remove any tests that referenced the deleted symbols.
5. Rollback: `git revert` the single commit — no data migration, no external callers.

## Open Questions

- None at design time.
