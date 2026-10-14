## Why

Add accounts as a resource to the YNAB HTTP MCP to provide LLMs with access to account information. This enables insights about account balances, types, and distribution across financial institutions. Currently, the MCP provides categories, planning, and transactions resources, but lacks account-level data which is essential for comprehensive financial analysis.

## What Changes

- Add a new `get_accounts` method to the `YnabService` class
- Create a new accounts tool module following the same pattern as categories
- Add accounts schemas for data validation and cleaning
- Register the accounts resource with the MCP server
- Create appropriate MCP resource endpoint for accounts data

## Capabilities

### New Capabilities
- `accounts`: Access to YNAB account information including account names, types, balances, and institution details

### Modified Capabilities
- None (this adds new functionality without changing existing requirements)

## Impact

- New file: `src/ynab_http_mcp/tools/accounts.py`
- New file: `src/ynab_http_mcp/schemas/accounts.py`
- Modified file: `src/ynab_http_mcp/ynab_service.py` (add get_accounts method)
- Modified file: `src/ynab_http_mcp/server.py` (import and register accounts tool)
- New MCP resource endpoint: `data://accounts`
