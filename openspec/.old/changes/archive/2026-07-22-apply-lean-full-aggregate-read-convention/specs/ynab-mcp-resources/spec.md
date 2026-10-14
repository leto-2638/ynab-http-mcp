## ADDED Requirements

### Requirement: Drill-in resources follow the `/{id}/full` URI convention

The system SHALL register a FastMCP resource template at `data://{entity}/{id}/full` for each read entity that has a single-entity endpoint: `data://categories/{category_id}/full`, `data://accounts/{account_id}/full`, `data://payees/{id}/full`, `data://transactions/{id}/full`, and `data://months/{ym}/full`. Each drill-in resource returns the corresponding `*Full` Pydantic model as JSON with MIME type `application/json`. The drill-in payload is the lean payload with one additional field, `full_details: dict`, containing the cleaned raw YNAB SDK object for the entity. The drill-in resource SHALL NOT omit, abbreviate, or rearrange any field present in the corresponding single-entity lean endpoint.

#### Scenario: Drill-in resources are discoverable
- **WHEN** an MCP client calls `list_mcp_resource_templates()`
- **THEN** the response includes templates at `data://categories/{category_id}/full`, `data://accounts/{account_id}/full`, `data://payees/{id}/full`, `data://transactions/{id}/full`, `data://months/{ym}/full`
- **AND** each template's description documents that the response includes the lean fields plus `full_details`

#### Scenario: Drill-in response includes all lean fields
- **WHEN** the LLM reads `data://categories/{category_id}/full` for a category with id, name, formatted budget/activity/balance, and a goal
- **THEN** the response JSON contains the lean `MCPCategory` fields (id, category_group_id, name, hidden, internal, deleted, budgeted_formatted, activity_formatted, balance_formatted)
- **AND** the lean `MCPCategoryGoal` is nested under `goal`
- **AND** the response additionally contains a top-level `full_details` field with the cleaned raw `ynab.Category` as a dict

#### Scenario: Drill-in response is bigger than the lean response
- **WHEN** the same entity is fetched via the lean URI and the full URI
- **THEN** the full URI response's JSON length is strictly greater than the lean URI's
- **AND** the difference is accounted for by the added `full_details` field

### Requirement: Transactions aggregate resource is registered

The system SHALL register a FastMCP resource template at `data://transactions/insights` returning a `TransactionInsightsResponse` JSON payload. The resource template SHALL accept optional query parameters `since_date` (ISO 8601, inclusive), `until_date` (ISO 8601, exclusive), and `account_id` (UUID). When no `since_date` and no `until_date` are provided, the server SHALL use the last 3 calendar months as the default window (current month and the previous two). The full behavioural contract is defined in the `transaction-aggregate-resource` capability spec.

#### Scenario: Aggregate resource is discoverable
- **WHEN** an MCP client calls `list_mcp_resource_templates()`
- **THEN** the response includes a template at `data://transactions/insights`
- **AND** the template's description references the default 3-month window and the supported query parameters

#### Scenario: Aggregate resource accepts time window parameters
- **WHEN** the LLM reads `data://transactions/insights?since_date=2024-01-01&until_date=2024-04-01`
- **THEN** the server fetches transactions in that window via `ynab_service.get_transactions(since_date="2024-01-01", until_date="2024-04-01", type="all")`
- **AND** the response's `monthly_buckets` covers 2024-01, 2024-02, 2024-03

#### Scenario: Aggregate resource default window is last 3 months
- **WHEN** the LLM reads `data://transactions/insights` with no parameters and the current date is 2025-03-15
- **THEN** the server fetches transactions from 2024-12-01 through 2025-04-01
- **AND** the response covers exactly 3 calendar months: the current month and the previous two (e.g. on 2025-03-15 the response covers 2025-01, 2025-02, 2025-03) — the exact boundary convention is defined in the `transaction-aggregate-resource` spec
- **AND** `monthly_buckets` contains exactly 3 entries, one per covered month (including the current month, even if partial)

### Requirement: Lean resources never embed `full_details` or aggregate data

The system SHALL NOT include `full_details` or aggregate data on any lean list resource (`data://categories`, `data://accounts`, `data://payees`, `data://transactions`, `data://months`, `data://months/{ym}`, or any per-entity-collection URI). The three layers (Lean / Full / Aggregate) live at distinct URIs and never co-occur in one response payload.

#### Scenario: Lean list responses are unchanged in shape
- **WHEN** the LLM reads `data://categories`, `data://accounts`, `data://payees`, `data://transactions`, or `data://months`
- **THEN** the response contains no `full_details` field anywhere
- **AND** the response contains no `monthly_buckets`, `top_payees`, `top_categories`, or other aggregate fields
- **AND** the response contains no `insights` or `aggregate` sub-object
