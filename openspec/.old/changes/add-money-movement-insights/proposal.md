## Why

The server currently exposes pre-computed spending insights (transactions, top payees, trend) but no parallel view for **budget-side planning quality**. Money movements — when a user reshuffles dollars between categories mid-month — are the clearest signal that the plan diverged from reality. Without an aggregate view, the LLM only sees raw rows and cannot tell the user *their budget is reactive* or *Groceries keeps subsidising Dining Out*. This change gives the LLM a single read that summarises planning-quality over a window, designed to be combined with the existing spending insights to give concrete guidance.

## What Changes

- New service method `YnabService.get_money_movements(month)` that wraps `ynab.MoneyMovementsApi.get_money_movements_by_month`. The service iterates one SDK call per month in the requested window (no date filtering; bounded load).
- New schema module `schemas/money_movement_aggregate.py` defining `MoneyMovementInsightsResponse` and supporting row models. Serialised with `exclude_none=True`, following the Lean / Full / Aggregate convention (this is the second Aggregate-layer entry, matching the existing `data://transactions/insights` aggregate resource).
- New tool module `tools/money_movements.py` registering two FastMCP read-only tools:
  - `get_money_movement_insights(since_date?, until_date?)` — window-based view, default last 3 calendar months.
  - `get_money_movement_insights_for_month(month_date)` — single-month drill-in.
- Output includes a derived `planning_health` label (`excellent` / `healthy` / `mixed` / `reactive`) so the LLM has one-glance read access, plus the underlying counts so it can recompute.
- Zero-movement windows return `planning_health: "excellent"` and a near-empty response (no fabricated rows).
- Server registration in `server.py`. New unit tests in `tests/test_money_movement_aggregate.py` plus an entry in `tests/test_mcp_catalog_footprint.py` for URI registration.

## Capabilities

### New Capabilities

- `money-movement-aggregate-tool`: pre-computed insights over a window of money movements — volume, mix (TBA vs other-category sources), donor/recipient top-N, recurring source→destination pairs, monthly trend, and a `planning_health` summary. Default window is the last 3 calendar months. Mirrors the `transaction-aggregate-resource` capability shape, but exposed as FastMCP tools (`get_money_movement_insights`, `get_money_movement_insights_for_month`) alongside the existing insight tools (`check_budget_health`, `get_spending_insights`).

### Modified Capabilities

- `ynab-service-layer`: adds `get_money_movements(month)` returning `ynab.MoneyMovementsResponse`. Month-by-month iteration is the only windowing strategy; no `since_date` / `until_date` API call path.
- `mcp-context-efficiency`: no behavioural requirement change, but the new tools must respect the Lean / Full / Aggregate convention already documented there (no full_details in aggregate, exclude_none on serialisation).

## Impact

- **Code**: `src/ynab_http_mcp/ynab_service.py` (+1 method), `src/ynab_http_mcp/schemas/money_movement_aggregate.py` (new), `src/ynab_http_mcp/tools/money_movements.py` (new), `src/ynab_http_mcp/server.py` (+1 register call).
- **SDK**: only existing `ynab.MoneyMovementsApi` methods; no dependency upgrades.
- **HTTP load**: for a default 3-month window the tool issues 3 SDK calls (one per month). A user-initiated 12-month view issues 12 calls. The aggregate builder does no extra round-trips.
- **Tests**: new unit tests for the builder, new footprint entry, no live-API integration tests.
- **Lint / type**: passes existing mypy strict and ruff config.
