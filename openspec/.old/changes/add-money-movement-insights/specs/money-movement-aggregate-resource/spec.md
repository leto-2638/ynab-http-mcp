# money-movement-aggregate-tool Specification

## Purpose

TBD - created by archiving change add-money-movement-insights. Update Purpose after archive.

## ADDED Requirements

### Requirement: `get_money_movement_insights` tool is registered

The system SHALL register a FastMCP tool named `get_money_movement_insights` annotated with `readOnlyHint=True`. The tool accepts two optional string parameters: `since_date` (ISO 8601, inclusive) and `until_date` (ISO 8601, exclusive). It returns a `MoneyMovementInsightsResponse` Pydantic model whose `model_dump_json()` representation has MIME type `application/json`. The tool SHALL be discoverable via `list_mcp_tools()`.

#### Scenario: Tool is registered on server start

- **WHEN** the server module is imported and `main()` is called
- **THEN** `list_mcp_tools()` returns an entry named `get_money_movement_insights`
- **AND** the tool's `description` describes the time-window parameters
- **AND** the tool's `annotations.readOnlyHint` is `true`

#### Scenario: Default time window is last 3 calendar months

- **WHEN** the LLM calls `get_money_movement_insights` with no `since_date` or `until_date`
- **THEN** the server computes `since_date` as the first day of (current month − 2 months)
- **AND** `until_date` is the first day of the month after the current month
- **AND** the response covers exactly 3 calendar months, including the current month and the previous two

#### Scenario: Custom time window is honoured

- **WHEN** the LLM calls `get_money_movement_insights(since_date="2025-04-01", until_date="2025-07-01")`
- **THEN** the response's `monthly_buckets` contain entries for 2025-04, 2025-05, and 2025-06 only
- **AND** `period_start == "2025-04-01"` and `period_end == "2025-07-01"`

#### Scenario: Invalid dates return an error response

- **WHEN** the LLM calls `get_money_movement_insights(since_date="not-a-date")`
- **THEN** the response contains an `error` key
- **AND** the error message indicates the invalid date format

### Requirement: `get_money_movement_insights_for_month` per-month drill-in tool is registered

The system SHALL register a FastMCP tool named `get_money_movement_insights_for_month` annotated with `readOnlyHint=True`. The tool accepts one required string parameter `month_date` (YYYY-MM or YYYY-MM-DD) and returns a `MoneyMovementInsightsResponse` Pydantic model covering exactly one month.

#### Scenario: Single-month tool returns exactly one bucket

- **WHEN** the LLM calls `get_money_movement_insights_for_month(month_date="2025-06")`
- **THEN** `monthly_buckets` contains exactly one entry with `month == "2025-06"`
- **AND** `period_start == "2025-06-01"` and `period_end == "2025-07-01"`
- **AND** the response is derived from a single SDK call (`get_money_movements_by_month(plan_id, month)`)

#### Scenario: Invalid month_date returns an error response

- **WHEN** the LLM calls `get_money_movement_insights_for_month(month_date="not-a-month")`
- **THEN** the response contains an `error` key
- **AND** the error message indicates the invalid month format

### Requirement: `MoneyMovementInsightsResponse` schema is defined

The system SHALL return a `MoneyMovementInsightsResponse` containing:

- `period_start: str` — ISO date of the first day of the analysis window
- `period_end: str` — ISO date of the day after the last day of the analysis window
- `monthly_buckets: List[MonthlyMoneyMovementBucket]` — one bucket per calendar month in the window (zero-filled if no movements)
- `total_movement_count: int` — number of money-movement rows in the window
- `total_moved_formatted: str` — absolute sum of `amount` over the window, YNAB-formatted
- `average_movement_formatted: str` — `total / total_movement_count`, YNAB-formatted (or `"$0.00"` if no movements)
- `from_ready_to_assign_count: int` — count of movements where `from_category_id is None`
- `from_ready_to_assign_total_formatted: str` — sum of `amount` over those movements, YNAB-formatted
- `from_other_categories_count: int` — count of movements where `from_category_id` is set
- `from_other_categories_total_formatted: str` — sum of `amount` over those movements, YNAB-formatted
- `top_source_categories: List[SourceCategoryAggregate]` — top 5 donor categories by absolute total milliunits descending; TBA (`from_category_id is None`) is reported as a single aggregate named `"Ready to Assign"`
- `top_destination_categories: List[DestinationCategoryAggregate]` — top 5 recipient categories by absolute total milliunits descending
- `recurring_pairs: List[RecurringMovementPair]` — pairs of `(from_category_id, to_category_id)` appearing at least twice in the window
- `movement_trend: Literal["increasing", "decreasing", "stable"]` — directional read of `total_moved_formatted` over `monthly_buckets`
- `proactive_pct: Optional[float]` — fraction of movements whose `moved_at` day-of-month is ≤ 7; `None` when `total_movement_count == 0`
- `planning_health: Literal["excellent", "healthy", "mixed", "reactive"]` — derived label
- `error: Optional[str]` — populated when computation fails

#### Scenario: monthly_buckets are zero-filled

- **WHEN** the window includes 3 months and only 1 month has movements
- **THEN** `monthly_buckets` contains 3 entries
- **AND** the empty months have `movement_count == 0`, `from_tba_count == 0`, `from_other_count == 0`, `from_tba_total_formatted == "$0.00"`, `from_other_total_formatted == "$0.00"`

#### Scenario: planning_health is "excellent" when there are no movements

- **WHEN** the window contains zero money movements
- **THEN** `total_movement_count == 0`
- **AND** `planning_health == "excellent"`
- **AND** `from_ready_to_assign_count == 0`
- **AND** `from_other_categories_count == 0`
- **AND** `top_source_categories == []`
- **AND** `top_destination_categories == []`
- **AND** `recurring_pairs == []`
- **AND** `monthly_buckets` still contains one zero-filled entry per month in the window
- **AND** `proactive_pct is None`

#### Scenario: planning_health is "reactive" when TBA ratio is low or pairs recur

- **WHEN** more than half of the movements in the window are sourced from non-TBA categories
- **THEN** `planning_health == "reactive"`
- **WHEN** `recurring_pairs` contains 3 or more entries
- **THEN** `planning_health == "reactive"` regardless of TBA ratio

#### Scenario: planning_health is "healthy" or "mixed" otherwise

- **WHEN** at least 80% of movements are sourced from TBA **and** `recurring_pairs` is empty
- **THEN** `planning_health == "healthy"`
- **WHEN** 50% ≤ TBA ratio < 80% **or** exactly one recurring pair exists (without the reactive thresholds above)
- **THEN** `planning_health == "mixed"`

#### Scenario: top_source_categories excludes single-occurrence TBA pairs

- **WHEN** the window contains 10 movements sourced from 5 distinct (non-TBA) categories plus 5 movements sourced from TBA
- **THEN** `top_source_categories` contains at most 5 entries
- **AND** the entries are sorted by `total_milliunits` absolute value descending
- **AND** a TBA-aggregated entry with `category_name == "Ready to Assign"` is included if it ranks in the top 5 by absolute total

#### Scenario: top_destination_categories truncates to 5

- **WHEN** the window contains movements targeting 12 distinct categories
- **THEN** `top_destination_categories` contains exactly 5 entries
- **AND** the entries are sorted by `total_milliunits` descending

#### Scenario: recurring_pairs excludes one-off pairs

- **WHEN** a `(from_category_id, to_category_id)` tuple appears exactly once in the window
- **THEN** it does not appear in `recurring_pairs`
- **WHEN** the same tuple appears at least twice
- **THEN** it appears in `recurring_pairs` exactly once with `movement_count` equal to its occurrence count and `total_formatted` equal to its summed amount

#### Scenario: movement_trend applies the same classification rule as spending_trend

- **WHEN** `monthly_buckets` shows total moved amount growing month over month
- **THEN** `movement_trend == "increasing"`
- **WHEN** `monthly_buckets` shows total moved amount shrinking
- **THEN** `movement_trend == "decreasing"`
- **WHEN** the absolute change across buckets is below the threshold
- **THEN** `movement_trend == "stable"`

#### Scenario: YNAB API failure produces an error response

- **WHEN** the underlying YNAB API call fails (network, auth, rate limit)
- **THEN** the response contains `error: str` describing the failure
- **AND** every numeric field is exactly `0` (or `"$0.00"` for formatted strings)
- **AND** every list field is exactly `[]`
- **AND** `monthly_buckets` contains exactly one zero-filled entry per month in the requested window
- **AND** the LLM can pattern-match `if response.error: ...` without worrying about partial data

### Requirement: Supporting schemas are defined

The system SHALL define the following supporting Pydantic models with the fields described.

`MonthlyMoneyMovementBucket`:
- `month: str` — YYYY-MM
- `movement_count: int`
- `from_tba_count: int`
- `from_other_count: int`
- `from_tba_total_formatted: str`
- `from_other_total_formatted: str`
- `average_moved_day_of_month: Optional[float]` — `None` if `movement_count == 0`

`SourceCategoryAggregate`:
- `category_id: Optional[str]` — `None` for the TBA aggregate
- `category_name: str` — `"Ready to Assign"` for the TBA aggregate
- `movement_count: int`
- `total_milliunits: int` — signed (negative = outflow from the source); sort key
- `total_formatted: str`
- `average_milliunits: int`

`DestinationCategoryAggregate`:
- `category_id: str`
- `category_name: str`
- `movement_count: int`
- `total_milliunits: int` — positive; sort key
- `total_formatted: str`
- `average_milliunits: int`

`RecurringMovementPair`:
- `from_category_id: Optional[str]`
- `from_category_name: str` — `"Ready to Assign"` for `None`
- `to_category_id: str`
- `to_category_name: str`
- `movement_count: int`
- `total_formatted: str`

#### Scenario: All supporting schemas are importable

- **WHEN** `from ynab_http_mcp.schemas.money_movement_aggregate import MonthlyMoneyMovementBucket, SourceCategoryAggregate, DestinationCategoryAggregate, RecurringMovementPair, MoneyMovementInsightsResponse` is executed
- **THEN** no `ImportError` is raised
- **AND** each model can be instantiated with its documented fields

### Requirement: Windowing is implemented month-by-month via one SDK call per month

The system SHALL define a `YnabService.get_money_movements(since_date: Optional[str], until_date: Optional[str])` method that walks the calendar months in the half-open `[since_date, until_date)` interval and calls `ynab.MoneyMovementsApi.get_money_movements_by_month(plan_id, month)` exactly once per month. The method SHALL return the concatenated `ynab.MoneyMovement` list to the caller. For a 3-month default window this SHALL be 3 SDK calls. The YNAB `since_date` / `until_date` query parameters SHALL NOT be used (the SDK endpoint does not accept them).

#### Scenario: Default window issues exactly 3 SDK calls

- **WHEN** the LLM calls `get_money_movement_insights` with no `since_date` or `until_date`
- **THEN** the service issues exactly 3 SDK calls — one for each of the 3 months in the default window

#### Scenario: Single-month drill-in issues exactly 1 SDK call

- **WHEN** the LLM calls `get_money_movement_insights_for_month(month_date="2025-06")`
- **THEN** the service issues exactly 1 SDK call (`get_money_movements_by_month` for that month)

### Requirement: Category-name resolution cross-references the categories endpoint

The system SHALL resolve `from_category_name` and `to_category_name` for each movement by consulting `ynab_service.get_categories()` once per `get_money_movement_insights` call. If the categories call fails, every movement SHALL still be reported, with `category_name` falling back to `"Unknown"`.

#### Scenario: Successful names resolution

- **WHEN** the categories endpoint returns a category whose id matches a movement's `from_category_id`
- **THEN** the resulting `SourceCategoryAggregate.category_name` equals that category's `name`

#### Scenario: Categories endpoint failure produces "Unknown" names

- **WHEN** the categories endpoint raises an exception
- **THEN** the aggregate is still returned
- **AND** every `SourceCategoryAggregate.category_name` and `DestinationCategoryAggregate.category_name` not pre-resolved is the literal string `"Unknown"`
- **AND** `recurring_pairs` carry `"Unknown"` for any unresolved name
- **AND** `error` is **not** populated (this is a soft failure; the LLM is told via the names)

### Requirement: Aggregate tool output omits null Optional fields

The system SHALL serialise `MoneyMovementInsightsResponse` to the wire format using `model_dump_json(exclude_none=True)`. The aggregate tool SHALL NOT embed `full_details` and SHALL NOT be returned from any lean read resource.

#### Scenario: Wire response omits null Optional fields

- **WHEN** the LLM calls `get_money_movement_insights` on a window with no movements
- **THEN** the JSON response does not contain the literal string `"error": null`
- **AND** `monthly_buckets` is present as a list of zero-filled entries
- **AND** the response shape matches the Lean / Full / Aggregate convention

#### Scenario: Aggregate does not embed full_details

- **WHEN** the LLM calls `get_money_movement_insights`
- **THEN** no inner object carries a `full_details` key
- **AND** the response stays within the budget for an Aggregate read (top-N rows, not full SDK objects)
