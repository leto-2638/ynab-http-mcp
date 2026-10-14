## Context

The YNAB HTTP MCP currently provides access to categories, planning, and transactions data but lacks account-level information. Accounts are fundamental financial entities in YNAB that represent bank accounts, credit cards, cash, etc. Adding accounts as a resource will enable comprehensive financial analysis and insights.

## Goals / Non-Goals

**Goals:**
- Provide LLMs with access to YNAB account information
- Follow the same architectural patterns as existing resources (categories, planning, transactions)
- Maintain consistency with the codebase's error handling and data validation approach
- Enable insights about account balances, types, and distribution

**Non-Goals:**
- Account creation/modification (read-only access only)
- Real-time account balance updates
- Transaction-level operations (already covered by transactions resource)

## Decisions

### Follow Existing Resource Pattern
**Decision**: Implement accounts resource following the exact same pattern as categories resource
**Rationale**: 
- Maintains codebase consistency and predictability
- Reuses existing utilities (clean_ynab_data, simple_validate)
- Leverages proven error handling patterns
- Makes the codebase easier to maintain and understand

### Data Cleaning Approach
**Decision**: Use the same cleaning and validation utilities as other resources
**Rationale**: 
- Ensures consistent data quality across all resources
- Reuses battle-tested code
- Maintains the same simplified validation approach suitable for AI agents

### Schema Design
**Decision**: Create simplified Pydantic models mirroring the categories schema structure
**Rationale**: 
- Provides type safety and validation
- Follows the established pattern for simplified data models
- Makes the API response structure predictable and consistent

### Error Handling
**Decision**: Use the existing @handle_ynab_errors decorator
**Rationale**: 
- Consistent error handling across all YNAB API calls
- Proven reliability with existing resources
- Proper handling of 404 and other API errors

## Risks / Trade-offs

**[Risk] Account data structure differences**: The YNAB Accounts API might return different field structures than expected
→ **Mitigation**: Research the actual API response structure and create appropriate schemas

**[Risk] Performance impact**: Adding another resource could increase server startup time
→ **Mitigation**: Follow the same lazy-loading pattern as other resources

**[Risk] Schema validation failures**: Account data might contain unexpected fields or formats
→ **Mitigation**: Use the same fallback validation approach as categories resource

**[Risk] API rate limiting**: Additional API calls could hit YNAB rate limits
→ **Mitigation**: Reuse existing plan_id and configuration, which already handles rate limiting

## Migration Plan

**Deployment Steps:**
1. Add get_accounts() method to YnabService
2. Create accounts schemas
3. Create accounts tool module
4. Register accounts resource with MCP server
5. Test the new endpoint
6. Deploy to production server

**Rollback Strategy:**
- Simple code rollback since this is additive functionality
- No database migrations or breaking changes involved
- Can be safely removed if issues arise

## Open Questions

- What specific account fields are most useful for financial analysis?
- Should we include closed/deleted accounts in the response?
- What's the expected response time for the accounts endpoint?
