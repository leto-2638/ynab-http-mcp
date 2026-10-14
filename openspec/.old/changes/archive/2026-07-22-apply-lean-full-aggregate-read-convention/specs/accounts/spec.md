## ADDED Requirements

### Requirement: Account drill-in resource at `data://accounts/{account_id}/full`

The system SHALL register a FastMCP resource template at `data://accounts/{account_id}/full` that returns an `MCPAccountFull` JSON payload. The `MCPAccountFull` model SHALL inherit from the lean `MCPAccount` and SHALL add exactly one field, `full_details: dict`, containing the cleaned raw `ynab.Account` as a dict. The drill-in response SHALL include every field present in the lean `data://accounts/{account_id}` response plus the `full_details` field. The drill-in resource SHALL NOT be the default single-entity endpoint — the LLM is expected to read the lean endpoint first and drill in only when it needs SDK-fidelity data.

#### Scenario: Account drill-in resource is discoverable
- **WHEN** an MCP client calls `list_mcp_resource_templates()`
- **THEN** the response includes a template at `data://accounts/{account_id}/full`
- **AND** the template's description documents that the response includes lean fields plus `full_details`

#### Scenario: Account drill-in response includes full_details
- **WHEN** the LLM reads `data://accounts/{account_id}/full` for an account that has a `note`, a `direct_import_last_error`, an integer `balance` in milliunits, and a YNAB-side `cleared_balance` / `uncleared_balance` field
- **THEN** the response JSON contains the lean `MCPAccount` fields (id, name, type, on_budget, closed, deleted, formatted balance/cleared_balance/uncleared_balance, transfer_payee_id, last_reconciled_at, direct_import_linked, direct_import_in_error)
- **AND** the response additionally contains a top-level `full_details` field
- **AND** `full_details` includes `note`, integer milliunit fields, and any other fields the lean layer dropped (e.g. `interest_rate`, `available_balance`, `debt_escrow_amounts` if present in the raw response)

#### Scenario: Lean account response is unchanged
- **WHEN** the LLM reads `data://accounts/{account_id}` (the lean endpoint, not the full one)
- **THEN** the response JSON does not contain a `full_details` key
- **AND** the response is byte-for-byte the same shape as before this change
