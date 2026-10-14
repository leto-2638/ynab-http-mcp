## Why

The current transactions functionality is implemented as an MCP tool, which provides flexible filtering capabilities but doesn't follow the resource pattern used by other data endpoints (categories, accounts). Converting it to a resource will provide a more consistent API interface and enable better integration with MCP clients that expect resource-based data access patterns.

## What Changes

- Convert the `get_transactions` tool to a resource using the `@mcp.resource()` decorator
- Change the interface from returning a Pydantic model to returning a JSON string. Continue using the Pydantic schema validation internally.
- Implement a resource template with query parameters for filtering: `data://transactions{?since_date,until_date,type,account_id,month,payee_id,category_id}`
- Add validation logic to ensure at least one of `type`, `since_date`, or `until_date` filters is provided
- Update error handling to return proper JSON error responses
- Enhance schema with detailed field descriptions for better agent understanding
- Add contextual hints to response to explain complex YNAB-specific fields
- Maintain all existing filtering capabilities and data processing logic

## Capabilities

### New Capabilities
- `transactions-resource`: Resource-based access to transaction data with filtering capabilities

### Modified Capabilities
- None (this is a new capability, not modifying existing ones)

## Impact

- **API Interface**: Changes from tool-based to resource-based access pattern
- **Client Integration**: Existing clients using the tool interface will need to migrate to the new resource URI pattern
- **Code Structure**: Transactions module will follow the same pattern as categories and accounts modules
- **Error Handling**: Improved JSON error responses for validation failures
- **Performance**: No expected performance impact - same underlying data processing logic
