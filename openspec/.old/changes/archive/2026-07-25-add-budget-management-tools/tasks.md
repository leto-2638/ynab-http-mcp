## 1. Service Layer Extensions

- [x] 1.1 Add `update_month_category()` method to YnabService
- [x] 1.2 Add transaction creation methods to YnabService
- [x] 1.3 Add input validation for all new service methods
- [x] 1.4 Add error handling for write operations

## 2. Schema Layer

- [x] 2.1 Create request schema for update operations
- [x] 2.2 Create response schema for tool outputs
- [x] 2.3 Add validation methods for write requests
- [x] 2.4 Implement currency formatting utility with milliunits conversion
- [x] 2.5 Update all schemas to include formatted currency fields
- [x] 2.6 Add currency detection from YNAB plan settings

## 3. Tool Implementation

- [x] 3.1 Create budget_management.py tool module
- [x] 3.2 Implement `update_month_category` tool
- [x] 3.3 Implement `create_transaction` tool
- [x] 3.4 Implement `check_budget_health` tool
- [x] 3.5 Implement `get_spending_insights` tool
- [x] 3.6 Add proper MCP tool annotations

## 4. Server Integration

- [x] 4.1 Import and register new tool module in server.py
- [x] 4.2 Ensure proper error handling in server
- [x] 4.3 Add configuration for write operations

## 5. Testing

- [x] 5.1 Create unit tests for new service methods
- [x] 5.2 Create unit tests for new tools
- [x] 5.3 Create integration tests for complete workflows
- [x] 5.4 Test error conditions and edge cases

## 6. Documentation

- [x] 6.1 Update README with new tool descriptions
- [x] 6.2 Add usage examples for new tools
- [x] 6.3 Document error handling patterns
