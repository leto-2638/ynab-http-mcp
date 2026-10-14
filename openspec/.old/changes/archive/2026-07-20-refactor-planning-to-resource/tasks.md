## 1. Service Layer Extension

- [x] 1.1 Research YNAB SDK category methods for month-specific data
- [x] 1.2 Add `get_month_category` method to YnabService class
- [x] 1.3 Add proper error handling and validation for new service method
- [x] 1.4 Write unit tests for new service methods

## 2. Resource Conversion

- [x] 2.1 Convert `get_plan_month` from tool to resource at `data://months/{month_id}`
- [x] 2.2 Convert `get_all_plan_months` from tool to resource at `data://months`
- [x] 2.3 Add new resource `get_month_category_by_id` at `data://months/{month}/categories/{category_id}`
- [x] 2.4 Update all resource decorators with proper annotations
- [x] 2.5 Ensure day is always set to 01 for YNAB API compatibility

## 3. Schema Updates

- [x] 3.1 Review existing planning schemas for compatibility
- [x] 3.2 Create new schema for month-category response if needed
- [x] 3.3 Update schema validation in resource methods

## 4. Testing and Validation

- [x] 4.1 Test all new resource endpoints with valid inputs
- [x] 4.2 Test error cases (invalid months, non-existent categories)
- [x] 4.3 Verify response format consistency with existing endpoints
- [x] 4.4 Test backward compatibility if maintaining old tool endpoints

## 5. Documentation

- [x] 5.1 Update API documentation with new resource URIs
- [x] 5.2 Add examples for new month-category endpoint
- [x] 5.3 Create migration guide for existing clients
- [x] 5.4 Update README with new functionality

## 6. Cleanup

- [x] 6.1 Remove deprecated tool endpoints (if not maintaining backward compatibility)
- [x] 6.2 Run mypy and ruff to ensure code quality
- [x] 6.3 Verify all tests pass
- [x] 6.4 Update change status to complete
