## Context

The YNAB HTTP MCP server exposes five read resources (categories, transactions, accounts, payees, plan-months) plus several sub-resource variants (per-account transactions, per-payee transactions, per-month categories). Each is currently backed by a `MCPResponse` model that performs light validation on the YNAB SDK object but preserves the full raw field set — including milliunit twins of formatted currency strings, all 16 goal fields per category, and every passthrough field. The LLM is forced to compose plain-English summaries from these raw fields at every read, which costs reasoning tokens and produces inconsistent answers.

The same convention question ("how much should this read resource expose?") has come up in every new schema so far. Codifying a convention now means future schemas get it for free.

The transaction insights pattern already exists as `BudgetInsightsResponse` in `schemas/budget_management.py:155` (per-category historical analysis). The user wants one analogous resource for the whole transactions collection, scoped to a time window.

## Goals / Non-Goals

**Goals:**
- Define one consistent read-side convention (Lean / Full / Aggregate) and apply it to every read resource.
- Drop fields that the LLM can either ignore (milliunits when formatted exists), parse from derived strings, or recover via drill-in. Keep only fields that enable filter / sort / date math / unique lookup.
- Give the LLM a clean escape hatch (`full_details: dict`) for arithmetic and SDK-fidelity needs, without bloating the primary read path.
- Provide one aggregate resource for transactions — the most-queried collection — with pre-computed historical buckets, top-N breakdowns, and a directional trend.

**Non-Goals:**
- No aggregate resources for categories, accounts, payees, or months in this change. Easy to add later if needed.
- No changes to write tools, `MCPRequest` models, or any destructive operation. The convention is read-side only.
- No test framework scaffolding. Per `AGENTS.md`, no tests currently exist; this change adds the first unit tests for the convention but does not set up a new test runner. (pytest + pytest-asyncio are already in dev deps.)
- No new SDK methods, no new dependencies, no YNAB API changes.
- No CI configuration. None exists; not in scope.

## Decisions

### Decision 1: `*Full` models inherit from lean, do not curate

`MCPCategoryFull(MCPCategory)` plus one new field `full_details: dict`. The full model is *not* a hand-curated superset of the lean fields — it delegates the lean fields to the parent and adds only `full_details`. Rationale: a curated full model duplicates the lean shape and requires maintenance every time YNAB adds a field. A `dict` dump of `clean_ynab_data(raw.to_dict())` is future-proof and single-sourced.

Alternatives considered:
- **Embed `full_details` in the lean model.** Rejected: bloats every list response, defeats the lean layer's purpose.
- **Curate a `*Full` superset with each YNAB field typed.** Rejected: maintenance burden, naming conflicts with the lean layer's LLM-friendly names.

### Decision 2: Drill-in URIs use `/full` suffix, not query param

`data://categories/{id}/full` rather than `data://categories/{id}?full=true`. Rationale: MCP resource URIs are opaque to clients; selecting shape by query parameter is a tool pattern, not a resource pattern. The `/full` suffix matches the existing URI hierarchy (`{id}/transactions`, `{month_date}/categories/{category_id}`).

### Decision 3: `full_details` is a typed `dict`, not a Pydantic model

`full_details: dict` rather than a Pydantic model that mirrors the YNAB SDK. Rationale: the SDK already has a model. Wrapping it again means we maintain our mirror in addition to theirs. A cleaned `dict` (UUID→str, datetime→ISO) gives the LLM the data without the coupling. Pydantic serializes dicts cleanly.

### Decision 4: `MCPCategoryGoal` keeps exactly 3 raw fields

`goal_type` (filter), `goal_target_date` (date math), `goal_percentage_complete` (sort/compare). Plus 2 derived strings. Total: 5 fields. Rationale: each of the 3 raw fields enables a query that the derived strings cannot. Every other field either duplicates information already in the derived strings (`goal_target` vs `goal_summary`) or describes a niche attribute the LLM rarely needs (`goal_creation_month`, `goal_snoozed_at`, `goal_cadence_frequency`).

Alternatives considered:
- **Keep `goal_under_funded` / `goal_overall_left` milliunits for arithmetic.** Rejected: the LLM can drill into `full_details` for arithmetic. The lean layer is for comprehension, not computation.
- **Keep all 4 raw "type" fields (`goal_type`, `goal_needs_whole_amount`, `goal_cadence`, `goal_cadence_frequency`).** Rejected: only `goal_type` is needed for filtering. The rest belong in `goal_summary` prose.

### Decision 5: `MCPTransaction` drops `milli_amount`

The lean model exposes only the formatted `amount: Optional[str]`. The raw milliunit integer is recoverable from `full_details`. Rationale: the LLM never does arithmetic on transaction amounts during a read — when creating a transaction via tool, it sends the integer directly to `MCPRequest`; when reading, the formatted string is sufficient for display and human answer.

### Decision 6: `MonthCategory` kills `goal_technical_details: str`

The current `schemas/planning.py:64` field is a vestigial "I didn't decide" string. Replace it with the same nested lean `MCPCategoryGoal` the `MCPCategory` schema uses. Drop milliunit fields (`budgeted`, `activity`, `balance`) in favour of their formatted twins — `MCPCategory` already follows this pattern, so the planning schema aligns.

### Decision 7: Transactions aggregate uses last 3 months as default window

When the LLM reads `data://transactions/insights` without `since_date` / `until_date`, the server computes for the last 3 calendar months (current + previous 2). Rationale: gives `spending_trend` 3 data points (the minimum for a directional read), aligns with typical budget cycles, and bounds token cost.

`monthly_buckets` always includes the requested period's months even if empty (zero-filled), so the LLM can reason about "no spending in August" without re-requesting.

### Decision 8: Aggregate includes top-5 payees and top-5 categories by absolute amount

Borrowed from the user's `BudgetInsightsResponse` pattern (which surfaces top spend areas). Top-5 is enough to be actionable without bloating. Sorted by absolute value so both inflow-heavy and outflow-heavy payees show up.

### Decision 9: `spending_trend` is a literal enum, not a series

`Literal["increasing", "decreasing", "stable"]`. Computed server-side as a simple linear regression slope over the `monthly_buckets`. Rationale: the LLM should never have to compute "is my spending going up?" itself — that's exactly the kind of derived insight the aggregate exists for. If the LLM wants the raw series, it can drill into `full_details` per transaction.

### Decision 10: Convention is codified in `schemas/base.py` as a docstring

A short block at the top of `base.py` stating the three layers and the drop rule. Future schemas that need to expose new entities follow it without re-asking the convention question.

## Risks / Trade-offs

- **[Breaking change] `MCPCategoryGoal` field drop → any client that reads `goal_overall_left` / `goal_under_funded` / etc. fails.** Mitigation: these clients can drill into `data://categories/{id}/full` to recover the raw values. Document the breaking change in the proposal and in `AGENTS.md`.

- **[Breaking change] `MCPTransaction.milli_amount` drop → any client computing transaction sums fails.** Mitigation: same as above, plus the aggregate resource `data://transactions/insights` exposes pre-computed sums (so the common case is handled).

- **[Risk] Aggregate computation cost — 3 months of all transactions could be 1000+ rows.** Mitigation: YNAB's `since_date` filter is server-side, the per-month grouping is O(N) Python, and the result is small (~3 buckets + top-5 lists). For very heavy users, we add a `limit` parameter later. Not in v1.

- **[Risk] `full_details: dict` breaks type-checking clients that want typed fields.** Mitigation: by design — the LLM is the primary consumer of these resources, not typed Python clients. Documented in the design.

- **[Trade-off] `MonthCategory` becomes lean but loses fields the planning UI may need.** Mitigation: planning data is read by the LLM (not displayed in our UI). Drill-in via `data://months/{ym}/full` recovers the raw.

- **[Risk] `_explain_goal_type` and `_explain_goal_funding_status` may produce misleading strings when goal_type is "NEED" or unknown.** Mitigation: keep the existing test scenarios in the `category-resource-schema` spec. Add one new test that asserts the lean `MCPCategoryGoal` derived strings round-trip from the raw fields.

- **[Trade-off] No aggregate for categories / accounts / months.** Mitigation: user explicitly requested one aggregate only. If categories aggregate (e.g. "total underfunded across all goals") becomes a common need, it's a follow-up change.

## Migration Plan

This is a personal-use project with no production deployment pipeline. Migration is:

1. Implement behind the existing dev workflow (restart MCP server, smoke-test the new URIs against a real YNAB budget).
2. Update `AGENTS.md` to document the new convention and URIs in one place.
3. If any external consumer exists (none currently), notify them of the breaking field removals and the drill-in path.

No rollback strategy needed — the server restart is the rollback.

## Open Questions

- **Should `_explain_goal_type` and `_explain_goal_funding_status` be tightened** to drop boilerplate ("The goal is to...", "in the category")? My recommendation: yes, do it as a follow-up. Not blocking this change.
- **Should the `seasonal_pattern_detected: bool` from `BudgetInsightsResponse` be ported to the transactions aggregate?** It's a real signal (e.g. "your December spending is always 2x") but requires more computation. My recommendation: defer to a follow-up.
