## 1. YNAB Service Implementation

- [x] 1.1 Add get_accounts() method to YnabService class
- [x] 1.2 Import ynab.AccountsApi in ynab_service.py
- [x] 1.3 Test the new service method with a real YNAB API call

## 2. Accounts Schema Implementation

- [x] 2.1 Create CleanAccount Pydantic model with essential fields
- [x] 2.2 Create AccountsResponse model wrapping account list
- [x] 2.3 Register schemas with the global registry
- [x] 2.4 Add appropriate field descriptions and types

## 3. Accounts Tool Implementation

- [x] 3.1 Create accounts.py tool module following categories pattern
- [x] 3.2 Implement get_accounts MCP resource function
- [x] 3.3 Add data cleaning and validation logic
- [x] 3.4 Implement fallback response for validation failures
- [x] 3.5 Add proper error handling and debugging

## 4. Server Integration

- [x] 4.1 Import accounts tool in server.py
- [x] 4.2 Register accounts resource with MCP server
- [x] 4.3 Verify resource is available at data://accounts

## 5. Testing and Verification

- [x] 5.1 Test accounts endpoint with valid YNAB API key
- [x] 5.2 Test error handling for missing API key
- [x] 5.3 Test error handling for invalid plan
- [x] 5.4 Verify response structure matches schema
- [x] 5.5 Run mypy type checking
- [x] 5.6 Run ruff linting and formatting