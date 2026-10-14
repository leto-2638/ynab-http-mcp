## 1. Codify the convention in `schemas/base.py`

- [x] 1.1 Add a module-level docstring to `src/ynab_http_mcp/schemas/base.py` documenting the Lean / Full / Aggregate convention
- [x] 1.2 Document the drop rule: no milliunit twins of formatted strings; no formatted strings whose value is captured in a derived string; no fields not needed for filter / sort / date math

## 2. Refactor `MCPCategoryGoal` to the 5-field lean shape

- [x] 2.1 In `src/ynab_http_mcp/schemas/categories.py`, remove the 11 raw goal fields and the 4 formatted-twin fields from `MCPCategoryGoal` (keep only `goal_type`, `goal_target_date`, `goal_percentage_complete`, `goal_summary`, `goal_status`)
- [x] 2.2 Update the existing `_explain_goal_type` and `_explain_goal_funding_status` helpers if any refer to fields that are now removed (verify they only consume fields that remain or are removed by design)
- [x] 2.3 Update the existing category-resource-schema test scenarios if they assert on removed fields (or add a new test that asserts the lean model has exactly 5 fields)
- [x] 2.4 Fix the cadence math in `_explain_goal_type` for MF goals. Current bugs: weekly (cadence=2) with frequency=1 produces "every 2 weeks" (should be "every week"); yearly (cadence=13) with frequency=1 produces "every 13 years" (should be "every year"). Update the test in `tests/test_categories_schema.py:197-210` that locks in the wrong values, and add a regression test for biweekly (`cadence=2, frequency=2` → "every 2 weeks"). Verify against the live YNAB SDK semantics for cadence 0/1/2/3-12/13/14 before relying on the derived strings.

## 3. Add `MCPCategoryFull` and the categories drill-in resource

- [x] 3.1 In `src/ynab_http_mcp/schemas/categories.py`, add `MCPCategoryFull(MCPCategory)` with a single new field `full_details: dict` and a `from_ynab` classmethod that builds the lean fields via the existing path and assigns `full_details = clean_ynab_data(raw.to_dict())`
- [x] 3.2 In `src/ynab_http_mcp/tools/categories.py`, register a resource template at `data://categories/{category_id}/full` that returns the `MCPCategoryFull` JSON

## 4. Refactor `MCPTransaction` to drop `milli_amount` and add `MCPTransactionFull`

- [x] 4.1 In `src/ynab_http_mcp/schemas/transactions.py`, remove the `milli_amount: int` field from `MCPTransaction`
- [x] 4.1.1 In `src/ynab_http_mcp/schemas/transactions.py`, also remove `milli_amount: int` from `MCPSubTransaction` per the same "no milliunit twins" rule; the integer amount is reachable via `data://transactions/{id}/full` inside `full_details.subtransactions[i].amount`
- [x] 4.2 Update `MCPTransaction.from_ynab` to no longer pass `milli_amount=raw.amount` (it will still live in `full_details`)
- [x] 4.2.1 Update `MCPSubTransaction.from_ynab` to no longer pass `milli_amount=raw.amount` on each sub
- [x] 4.3 Add `MCPTransactionFull(MCPTransaction)` with `full_details: dict`
- [x] 4.4 In `src/ynab_http_mcp/tools/transactions.py`, register `data://transactions/{id}/full`
- [x] 4.5 Add a test asserting `MCPTransaction` has no `milli_amount` field, that `MCPSubTransaction` has no `milli_amount` field, and that `MCPTransactionFull.model_dump()` contains both the lean fields and the integer `amount` inside `full_details` (including inside `full_details.subtransactions`)
- [x] 4.6 Apply `exclude_none=True` to every lean `model_dump_json()` call across `src/ynab_http_mcp/tools/categories.py`, `accounts.py`, `payees.py`, `transactions.py`, `planning.py`; verify no `"key": null` appears in any lean response and the convention is documented in the data-models spec

## 5. Add `MCPAccountFull`, `CleanPayeeFull`, and the accounts / payees drill-in resources

- [x] 5.1 In `src/ynab_http_mcp/schemas/accounts.py`, add `MCPAccountFull(MCPAccount)` with `full_details: dict`
- [x] 5.2 In `src/ynab_http_mcp/tools/accounts.py`, register `data://accounts/{account_id}/full`
- [x] 5.3 In `src/ynab_http_mcp/schemas/payees.py`, add `CleanPayeeFull(CleanPayee)` with `full_details: dict`
- [x] 5.4 In `src/ynab_http_mcp/tools/payees.py`, register `data://payees/{id}/full`
- [x] 5.5 Update `PayeesResponse` / `PayeeResponse` to optionally return the Full shape (or add a sibling response class) — verify the existing tool wiring and pick the minimum-churn option

## 6. Refactor `MonthCategory` in the planning schema and add `PlanMonthFull`

- [x] 6.1 In `src/ynab_http_mcp/schemas/planning.py`, remove `goal_technical_details: Optional[str]` from `MonthCategory`
- [x] 6.2 In `MonthCategory`, drop the milliunit `budgeted`, `activity`, `balance` integer fields and rely on the `*_formatted` twins (align with `MCPCategory`)
- [x] 6.3 In `MonthCategory`, replace the flat goal fields with a nested `goal: Optional[MCPCategoryGoal]` (reuse the lean goal from the categories schema)
- [x] 6.4 Update `PlanMonth.from_ynab_response` to populate the new `goal` field via `MCPCategoryGoal.from_ynab`
- [x] 6.5 Add `PlanMonthFull(PlanMonth)` with `full_details: dict`
- [x] 6.6 In `src/ynab_http_mcp/tools/planning.py`, register `data://months/{ym}/full` and `data://months/{ym}/categories/{category_id}/full`

## 7. Build the transactions aggregate resource

- [x] 7.1 Create `src/ynab_http_mcp/schemas/transaction_aggregate.py` with `MonthlyTransactionBucket`, `PayeeAggregate`, `CategoryAggregate`, `ClearedBreakdown`, and `TransactionInsightsResponse` per the spec; `PayeeAggregate` and `CategoryAggregate` MUST include both `total_milliunits: int` (sort key) and `total_formatted: str` (display)
- [x] 7.2 In the same file, add a `TransactionInsightsResponse.from_transactions(transactions, period_start, period_end)` classmethod that computes all aggregates from a list of `ynab.TransactionDetail` objects
- [x] 7.3 Implement the `spending_trend` computation as a simple linear regression slope over outflow values; classify as `increasing` / `decreasing` / `stable` per a documented threshold
- [x] 7.4 In `src/ynab_http_mcp/tools/transactions.py`, register the resource template at `data://transactions/insights` accepting `since_date`, `until_date`, `account_id`
- [x] 7.5 Implement the default time window: when neither `since_date` nor `until_date` is provided, default to the last 3 calendar months (current + previous 2)
- [x] 7.6 Implement the zero-fill behaviour: `monthly_buckets` always has one entry per month in the window, with zeros when there are no transactions
- [x] 7.7 Implement the top-5 payee and top-5 category aggregation, with "Unassigned" / "Uncategorized" sentinels for null IDs
- [x] 7.8 Add unit tests for the aggregate computation: empty window, single month, multi-month, top-N truncation, unassigned payee / category, increasing / stable / decreasing trend, error path

## 8. Update documentation and convention reference

- [x] 8.1 Update `AGENTS.md` to document the Lean / Full / Aggregate convention with one example per layer
- [x] 8.2 Add the new resource URIs (`/full` and `/insights`) to the AGENTS.md resource list
- [x] 8.3 Add a short note in AGENTS.md flagging the breaking changes (`MCPCategoryGoal` field drops, `MCPTransaction.milli_amount` removal) and pointing at the drill-in path

## 9. Verification and hygiene

- [x] 9.1 `uv run mypy src` passes
- [x] 9.2 `uv run ruff check .` passes
- [x] 9.3 Start the server (`bash scripts/start-ynab-mcp.sh`) and read each new drill-in URI against a real YNAB budget; eyeball the JSON
- [x] 9.4 Read `data://transactions/insights` with and without query parameters; confirm the default 3-month window, the zero-fill behaviour, and the spending trend
- [x] 9.5 Confirm no lean list resource carries `full_details` anywhere in its payload
- [x] 9.6 Run all existing tests (if any) to confirm no regression in write tools or other read paths
