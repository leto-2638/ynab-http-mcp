## 1. Service layer

- [x] 1.1 Add `YnabService.get_money_movements(since_date, until_date)` that walks each calendar month in the half-open window and calls `ynab.MoneyMovementsApi.get_money_movements_by_month(plan_id, month)` once per month. Raise `ValueError` when `since_date` or `until_date` is missing / unparseable. Concatenate the per-month `data.money_movements` lists into one flat list and return it.

## 2. Aggregate schema

- [x] 2.1 Create `src/ynab_http_mcp/schemas/money_movement_aggregate.py` with the supporting models: `MonthlyMoneyMovementBucket`, `SourceCategoryAggregate`, `DestinationCategoryAggregate`, `RecurringMovementPair`, and the top-level `MoneyMovementInsightsResponse`.
- [x] 2.2 Reuse `_default_window`, `_months_in_window`, `_classify_trend`, `_ynab_format`, and `_extract_number` from `schemas/transaction_aggregate.py` rather than re-implementing.
- [x] 2.3 Define module constants `PROACTIVE_DAY_THRESHOLD = 7`, `_PLANNING_HEALTH_TBA_HEALTHY = 0.80`, `_PLANNING_HEALTH_TBA_MIXED = 0.50`, `_RECURRING_PAIR_HEALTHY_MAX = 0`, `_RECURRING_PAIR_REACTIVE_MIN = 3`, and a `_classify_planning_health(...)` helper that returns one of `"excellent"`, `"healthy"`, `"mixed"`, `"reactive"`.
- [x] 2.4 Implement `build_money_movement_insights(movements, period_start, period_end, category_name_lookup)` taking a flat list of `ynab.MoneyMovement`, the window dates, and a callable that maps `category_id → category_name`. Return a `MoneyMovementInsightsResponse`. Honour the `exclude_none=True` convention.
- [x] 2.5 Re-export the new symbols from `src/ynab_http_mcp/schemas/__init__.py` so `from ynab_http_mcp.schemas import MoneyMovementInsightsResponse` works.

## 3. MCP tool registration

- [x] 3.1 Create `src/ynab_http_mcp/tools/money_movements.py` with a `register(mcp, ynab_service)` function. Register two read-only FastMCP tools — `get_money_movement_insights(since_date?, until_date?)` (window view, default 3 months) and `get_money_movement_insights_for_month(month_date)` (single-month drill-in). Both return the `MoneyMovementInsightsResponse` Pydantic model directly.
- [x] 3.2 Both tools return `MoneyMovementInsightsResponse`. On any `ValueError` or SDK exception, return a fully zeroed `MoneyMovementInsightsResponse` with `error` populated. The framework serialises with `exclude_none=True` on the way out.
- [x] 3.3 Wire the new module into `src/ynab_http_mcp/server.py` via `money_movement_tools.register(mcp, ynab_service)`.

## 4. Tests

- [x] 4.1 Add `tests/test_money_movement_aggregate.py` mirroring the structure of `tests/test_transaction_aggregate.py`: window defaults, trend classification, zero-movement windows (planning_health == "excellent"), TBA vs other-category split, top-N truncation (donors and recipients), recurring-pair detection, proactive_pct computation, error path on inverted period.
- [x] 4.2 Register the two new URIs in `tests/test_mcp_catalog_footprint.py` alongside the existing entries so the server's resource template surface stays in lockstep.
- [x] 4.3 Run `uv run pytest tests/` and confirm green.

## 5. Lint and type checking

- [x] 5.1 Run `uv run ruff check src/ynab_http_mcp tests` and `uv run ruff format --check src/ynab_http_mcp tests`. Resolve any new findings.
- [x] 5.2 Run `uv run mypy src/ynab_http_mcp` and resolve any new findings.

## 6. Documentation

- [x] 6.1 Update `AGENTS.md` resource URI reference to add `data://money-movements/insights{?since_date,until_date}` and `data://months/{month_date}/money-movements/insights`.
- [x] 6.2 Add a short note to the Lean / Full / Aggregate aggregate examples mentioning the new aggregate is a second Aggregate-layer resource.
