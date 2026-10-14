## Why

Read-side MCP resources currently carry too much raw YNAB structure into the LLM context: milliunit fields alongside their formatted twins, all 16 goal attributes for every category, the full SDK payload wrapped twice. The LLM then has to do the composition work — turning 16 goal fields into "save $5k by Christmas, $340 underfunded" — which burns reasoning tokens, costs confidence, and inflates per-request spend. The user is also asking "what's the design principle here?" repeatedly as new schemas are added, which is a sign the convention needs to be codified.

## What Changes

- **Refactor `MCPCategoryGoal` from 16 raw fields to 5 fields** — keep `goal_type`, `goal_target_date`, `goal_percentage_complete` for filter / sort / date math; keep `goal_summary` and `goal_status` derived strings; drop the rest. **BREAKING** for any client that reads `goal_overall_left`, `goal_under_funded`, `goal_cadence_frequency`, etc. directly.
- **Refactor `MCPTransaction` to drop `milli_amount`** — formatted-only on the read side. **BREAKING** for any client reading the integer field; the value is recoverable from the `data://transactions/{id}/full` drill-in. The same rule applies to `MCPSubTransaction.milli_amount`, which is also dropped (subtransactions are nested inside the transaction, so no separate resource is needed).
- **Add a `*Full` sibling model per read entity** (categories, accounts, payees, transactions, months) that inherits from the lean model and adds one field, `full_details: dict`, containing the cleaned raw YNAB object.
- **Add drill-in resource templates** at `data://{entity}/{id}/full` that return the Full sibling.
- **Add one new aggregate resource** at `data://transactions/insights` returning a `TransactionInsightsResponse` — historical monthly buckets, top payees, top categories, cleared/uncleared breakdown, inflow/outflow, spending trend. Pattern stolen from `BudgetInsightsResponse`. Default window: last 3 calendar months.
- **Refactor `MonthCategory`** in the planning schema: kill `goal_technical_details: str`, drop milliunits, use the same nested lean `MCPCategoryGoal`.
- **Update `AGENTS.md`** to document the Lean / Full / Aggregate convention with examples.

## Capabilities

### New Capabilities
- `transaction-aggregate-resource`: defines the `data://transactions/insights` resource, its query parameters, default time window, and the `TransactionInsightsResponse` shape (monthly buckets, top-N breakdowns, spending trend).
- `read-resource-full-drill-in`: defines the `data://{entity}/{id}/full` pattern, the `*Full` model inheritance from lean, and the contract that `full_details` is the cleaned raw SDK object.

### Modified Capabilities
- `category-resource-schema`: `MCPCategoryGoal` shrinks from 16 raw fields to 5; the goal-summary derivation rules stay.
- `ynab-mcp-resources`: adds the `/full` drill-in templates and the new `data://transactions/insights` resource; documents the Lean / Full / Aggregate convention.
- `data-models`: documents the read-side model conventions (formatted-only amounts, no milliunits, no formatted duplicates, derived strings for opaque nested concepts).
- `accounts`: `MCPAccount` unchanged in shape but gains a `MCPAccountFull` sibling and a `data://accounts/{account_id}/full` resource.

## Impact

- 5 schema files in `src/ynab_http_mcp/schemas/` (categories, transactions, accounts, payees, planning) gain a `*Full` sibling and lose fields.
- 5 tool files in `src/ynab_http_mcp/tools/` (categories, transactions, accounts, payees, planning) gain a drill-in resource function and the transactions file gains the aggregate resource function.
- New schema module: `src/ynab_http_mcp/schemas/transaction_aggregate.py`.
- New tests in `tests/schemas/` and `tests/tools/`: unit tests for the `*Full` model inheritance, integration tests for the new resource URIs, and aggregate computation tests.
- `AGENTS.md` resource list extended with the new URIs and the convention description.
- No changes to `MCPRequest` (write side) or to any write tool.
- No new dependencies. No SDK changes. No CI changes.
