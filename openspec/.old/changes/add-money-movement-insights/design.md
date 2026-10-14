## Context

The server already aggregates transactions at `data://transactions/insights` — monthly buckets, top payees/categories, and a directional trend. The Aggregate layer in the Lean / Full / Aggregate convention is reserved for pre-computed views that turn raw rows into LLM-ready narratives. Today the Aggregate layer has exactly one resource.

Money movements are a separate YNAB data set (`ynab.MoneyMovementsApi`). Each row records a single category-to-category budget transfer inside a single month: `from_category_id`, `to_category_id`, `amount` (milliunits), `month`, `moved_at`. When `from_category_id` is `None`, the money was sourced from Ready to Assign (TBA) — a fresh assignment. When it is set, another category was sacrificed.

The user wants a parallel aggregate that surfaces **planning quality**: how often the plan had to be reshuffled, what fraction of reshuffles were TBA-sourced (proactive) versus category-sourced (reactive), which categories keep donating, which pairs recur, and whether the rate is improving or degrading. The LLM can then combine this with the transaction insights to give concrete advice ("you keep raiding Groceries — your Dining Out target is set too low").

The YNAB SDK exposes both `get_money_movements(plan_id)` (unscoped, no date filter) and `get_money_movements_by_month(plan_id, month)` (single month). The only windowing primitive is per-month, so the service layer iterates month-by-month using the by-month endpoint. This bounds HTTP load at exactly N calls for an N-month window and avoids pulling years of history.

## Goals / Non-Goals

**Goals:**
- Expose a windowed view of money-movement planning quality through two URIs that mirror the existing transaction aggregate URI pattern.
- Convert raw `ynab.MoneyMovement` rows into a single small response the LLM can quote verbatim: counts, formatted totals, top-N donors/recipients, recurring pairs, monthly trend, and a `planning_health` label.
- Stay inside the Lean / Full / Aggregate convention: the Aggregate lives at its own URI, is serialised with `exclude_none=True`, and is never embedded in lean resources.
- Reuse formatting and default-window helpers from `schemas/transaction_aggregate.py` rather than re-implementing them.

**Non-Goals:**
- Exposing individual `MoneyMovement` rows or a `*Full` drill-in. Raw movement data is not a read path the LLM needs — the aggregate captures everything actionable.
- Reactive-write tooling. This change does not add endpoints to move money; it only reads.
- Per-account or per-payee scoping. Movements are intra-budget operations; payees and accounts are out of scope.
- Per-month date filtering inside the SDK call. Date filtering is purely on the service side by walking months.

## Decisions

### D1. Month-by-month iteration in the service layer

The service method `get_money_movements(since_date, until_date)` iterates the calendar months in the half-open window `[since_date, until_date)` and calls `ynab.MoneyMovementsApi.get_money_movements_by_month(plan_id, month)` once per month. The SDK does not accept a date filter on the unscoped endpoint, and pulling the unscoped result for an old plan would inflate history unboundedly.

**Alternative considered**: one unscoped call + client-side `month`-filter. Rejected: unbounded history load for any reasonably-aged plan.

### D2. Two dedicated tools, default 3-month window

- `get_money_movement_insights(since_date?, until_date?)` — window view,
  defaults to last 3 calendar months when `since_date` and `until_date`
  are both absent.
- `get_money_movement_insights_for_month(month_date)` — single-month
  drill-in.

These are FastMCP **tools** (`@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))`)
following the same shape as the existing insight tools
(`check_budget_health`, `get_spending_insights`). They return the
`MoneyMovementInsightsResponse` Pydantic model directly, which gives the
LLM a typed `outputSchema` in the MCP catalog and removes the need for
the caller to parse a JSON string. Both tools are marked
`readOnlyHint=True` so they sit cleanly in the read-only side of the
catalog.

**Alternative considered**: a single URI with an optional `month` query
parameter (resource style). Rejected: the read-only insight tools in
this server (`check_budget_health`, `get_spending_insights`) are
already tools; adding a third insight as a resource would create an
inconsistent split between the two aggregate reads. Tool-style also
gives a typed `outputSchema` for free.

### D3. Planning-health label as a derived field, not a public threshold knob

`planning_health` is computed from the response's own counts using thresholds set in the builder module:

| Label | Rule |
|---|---|
| `excellent` | `total_movement_count == 0` |
| `healthy` | `from_tba_ratio >= 0.80` and `len(recurring_pairs) == 0` |
| `mixed` | `0.50 <= from_tba_ratio < 0.80`, or exactly one recurring pair |
| `reactive` | `from_tba_ratio < 0.50`, or `len(recurring_pairs) >= 3` |

The threshold constants live in `schemas/money_movement_aggregate.py` as module-level constants (e.g., `_PLANNING_HEALTH_TBA_HEALTHY = 0.80`). They are not exposed on the response model — future tuning should be a code change with new tests, not a query parameter.

**Alternative considered**: query-parameter thresholds. Rejected: deterministic results matter more than configurability for an insight tool — the LLM benefits from a stable definition of "reactive".

### D4. `proactive_day_threshold` as a builder argument, not a model field

A constant `PROACTIVE_DAY_THRESHOLD = 7` lives in `schemas/money_movement_aggregate.py`. The builder uses it to mark a movement as proactive (moved on day ≤ 7 of the month). The threshold is a builder arg so a future version can change it without breaking the response contract; the response only carries the *result* (`proactive_pct`).

**Alternative considered**: expose `proactive_day_threshold` on the response. Rejected: bloating the response with a knob value the LLM doesn't need.

### D5. Reuse `_default_window` and `_ynab_format` from `transaction_aggregate`

Both helpers are already implemented and tested. Re-importing them keeps currency formatting deterministic and avoids divergent locale handling between the two aggregates.

**Alternative considered**: re-implement inside `money_movement_aggregate.py`. Rejected: duplicated logic and divergent formatting over time.

### D6. Top-N truncation at 5, mirroring transactions

`top_source_categories` and `top_destination_categories` are capped at 5 rows, sorted by absolute total milliunits descending, matching `TransactionInsightsResponse`.

### D7. Recurring pairs flagged at count ≥ 2

A `recurring_pairs` row is included whenever the same `(from_category_id, to_category_id)` tuple appears at least twice in the window. Pairs with exactly one occurrence are excluded (every one-off move would otherwise pollute the list).

## Risks / Trade-offs

- **Sparse windows** → when the window contains zero movements, the response carries `total_movement_count: 0`, `planning_health: "excellent"`, empty lists and zero-formatted totals. The LLM can read this as a clean bill of health. [Risk: a user with very few movements could see "excellent" misleadingly on a 1-month window] → [Mitigation: the response still carries the explicit count so the LLM can caveat "0 movements in this short window" if it wishes; spec requires the count to be exposed verbatim].
- **Multi-month boundary drift** → a movement assigned to month 2025-06-01 could in principle have a `moved_at` outside that month (end-of-month rollover). [Risk: bucketing by `month` field instead of `moved_at` could misclassify timing signals] → [Mitigation: `proactive_pct` uses `moved_at` day-of-month, not the `month` field; bucketing for monthly totals still uses the API `month` since that is the budget-month, which is what the user cares about].
- **N HTTP calls for an N-month window** → a user-requested 12-month view issues 12 SDK calls. [Risk: a misuse that passes wide windows could be slow] → [Mitigation: default window is 3 months, and the spec caps the documented window to "the previous 6 months" guidance; no hard cap is required for personal use].
- **Category-name resolution** → the SDK's `MoneyMovement` model does not carry `from_category_name` or `to_category_name`. [Risk: the aggregate exposes category IDs without friendly names] → [Mitigation: the builder cross-references the categories endpoint once per call to resolve names; if the categories call fails, IDs fall back and names fall back to `"Unknown"`].
- **Currency template** → with zero movements in the window there is no `amount_formatted` to derive a template from. [Risk: formatted currency defaults to `"$0.00"` and may not match the plan's locale] → [Mitigation: accept the default dollar sign on empty windows; this is the same trade-off the transaction aggregate accepts].

## Open Questions

None — all design decisions captured above.
