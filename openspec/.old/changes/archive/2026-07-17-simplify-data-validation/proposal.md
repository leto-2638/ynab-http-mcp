## Why

The current data validation and transformation system is overly complex for the needs of this MCP. Multiple validation passes, custom error handling, and complex schema configurations create unnecessary overhead while providing minimal benefit for the simple data cleaning required to make YNAB API responses agent-friendly.

## What Changes

- **Simplify data validation**: Replace multi-layered validation with single-pass cleaning and basic validation
- **Unified data cleaning**: Create a single `clean_ynab_data()` function to handle all data transformation needs
- **Simplified schemas**: Replace complex Pydantic models with simpler ones using basic data types
- **Remove redundant validation**: Eliminate duplicate validation passes and custom error handling
- **Update tests**: Remove tests for complex validation, keep integration tests with simplified approach

## Capabilities

### New Capabilities
- `data-cleaning`: Unified data cleaning function for YNAB API responses
- `simple-validation`: Basic validation approach using simplified Pydantic models
- `agent-friendly-data`: Ensure all tool responses use simple data types suitable for agents

### Modified Capabilities
- `ynab-service-layer`: Update to use simplified data cleaning approach
- `data-models`: Simplify existing data models and validation requirements

## Impact

- **Code**: Significant reduction in complexity across tools, schemas, and validation logic
- **Tests**: Removal of complex validation tests, simplification of remaining tests
- **Performance**: Improved response times due to fewer validation passes
- **Maintainability**: Much easier to understand and modify the data processing pipeline
- **Agent Compatibility**: Better suited for agent consumption with simpler data types