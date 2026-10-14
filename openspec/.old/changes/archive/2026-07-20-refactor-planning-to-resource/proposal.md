## Why

The current `planning.py` module uses `@mcp.tool()` decorators while other modules like `categories.py` use `@mcp.resource()` decorators. This inconsistency makes the API surface uneven and harder to maintain. Converting planning tools to resources will create a more consistent architecture and enable better resource-oriented patterns.

## What Changes

- Convert `get_plan_month` tool to a resource at `data://months/{month_id}`
- Convert `get_all_plan_months` tool to a resource at `data://months`
- Add new resource `get_month_category_by_id` at `data://months/{month}/categories/{category_id}`
- Add corresponding YNAB service methods to support the new category endpoint

## Capabilities

### New Capabilities
- `month-resource`: Expose month planning data as MCP resources instead of tools
- `month-category-lookup`: Retrieve category details for a specific month

### Modified Capabilities
- None (this is a pure refactoring with no breaking changes to existing functionality)

## Impact

- `src/ynab_http_mcp/tools/planning.py` - Complete refactoring from tools to resources
- `src/ynab_http_mcp/ynab_service.py` - New method additions for category lookup
- `src/ynab_http_mcp/schemas/planning.py` - Potential schema updates for new response types
- API consumers will need to update their URIs from tool paths to resource paths
