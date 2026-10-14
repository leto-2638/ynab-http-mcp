## Context

The current YNAB HTTP MCP provides read-only access to YNAB budget data through MCP resources. The system uses a clean three-layer architecture:
- Service layer (`ynab_service.py`) handles API calls with validation
- Schema layer (`schemas/`) provides Pydantic models for response transformation
- Tool layer (`tools/`) registers MCP resources for data access

The budgeting advice agent currently can only read data but cannot make changes. This design extends the architecture to support write operations while maintaining the existing patterns and safety.

## Goals / Non-Goals

**Goals:**
- Enable atomic money movement between categories
- Provide computed spending insights for better decision making
- Detect and prevent budget overassignment
- Maintain existing read-only functionality
- Follow YNAB API best practices for write operations
- Provide clear error handling and validation

**Non-Goals:**
- Replace YNAB's native audit logging (YNAB already handles this)
- Support bulk operations (focus on single atomic operations)
- Implement complex financial planning algorithms
- Add user interface components

## Decisions

### 1. Specialized Tools vs Generic Write Operations
**Decision**: Create specialized tools (`reassign_money`, `check_budget_health`, `get_category_spending_insights`) rather than exposing generic write operations.

**Rationale**: 
- Specialized tools provide better safety through built-in validation
- Atomic operations reduce error potential
- Matches the agent's use case of providing budgeting advice
- Hides complexity of implementing money movement via transactions

**Alternatives Considered**:
- Generic write tools (rejected due to safety concerns)
- Direct exposure of YNAB write endpoints (rejected as too low-level)

### 2. Tool vs Resource Registration
**Decision**: Register write operations as MCP `tools` (not `resources`) with appropriate annotations (`destructiveHint: True`).

**Rationale**:
- MCP conventions specify tools for non-idempotent, destructive operations
- Resources are for idempotent, read-only data access
- Clear semantic signaling to clients about operation characteristics

### 3. Money Movement Implementation
**Decision**: Implement `reassign_money` using YNAB transaction creation rather than direct category updates.

**Rationale**:
- YNAB API doesn't have direct "move money" endpoint
- Transaction-based approach provides proper audit trail
- Matches YNAB's internal money movement model
- More reliable than trying to update two categories separately

### 4. Validation Strategy
**Decision**: Implement multi-level validation: input validation in tools, business logic validation in service layer, and response validation via schemas.

**Rationale**:
- Defense in depth approach
- Clear separation of concerns
- Consistent with existing read-only patterns
- Provides helpful error messages at each layer
### 5. Error Handling Pattern

**Decision**: Return structured JSON error objects rather than raising exceptions.

**Rationale**:
- Consistent with existing MCP resource error handling
- Easier for agents to parse and handle errors
- Provides machine-readable error information
- Maintains compatibility with MCP client expectations

### 6. Budget Insights Implementation Strategy

**Decision**: Implement `get_budget_insights` as a comprehensive cross-month analysis tool that leverages existing YNAB API fields (activity, budgeted, balance, goal progress) rather than parsing transactions.

**Rationale**:
- YNAB API already provides rich budget data through `get_month_category()` endpoint
- Activity field contains pre-computed spending data, eliminating need for transaction parsing
- Goal-related fields provide built-in progress tracking
- Leverages existing schema infrastructure (MonthCategory)
- More performant and reliable than transaction-based analysis

**Alternatives Considered**:
- Transaction-based analysis (rejected due to complexity and performance)
- Simple trend calculation (rejected as not comprehensive enough)
- External data processing (rejected to maintain system simplicity)

### 7. Currency Format Standardization

**Decision**: Use YNAB API's built-in `_formatted` properties directly, eliminating the need for custom currency formatting logic.

**Rationale**:
- YNAB API already provides perfectly formatted currency strings
- Formatted fields include correct localization, symbols, and decimal places
- Eliminates complex custom formatting code
- Ensures consistency with YNAB's official formatting
- Zero maintenance burden

**Implementation Strategy**:

**Leverage YNAB Formatted Fields**:
- YNAB API responses include `_formatted` versions of all monetary fields
- These fields already contain properly formatted currency strings
- Example: `budgeted_formatted`: "£500.00", `activity_formatted`: "-£450.00"

**Schema Updates Required**:
1. **Replace raw fields with formatted equivalents**:
   - Remove milliunit fields (budgeted, activity, balance, amount, etc.)
   - Use corresponding `_formatted` fields from YNAB API
   - Apply consistently across all schemas

2. **Update all affected schemas**:
   - `CleanCategory`: `budgeted` → `budgeted_formatted`, `balance` → `balance_formatted`, `activity` → `activity_formatted`
   - `MonthCategory`: `budgeted` → `budgeted_formatted`, `activity` → `activity_formatted`, `balance` → `balance_formatted`
   - `Transaction`: `amount` → `amount_formatted`
   - `Account`: `balance` → `balance_formatted`
   - All other monetary fields in any schema

**Field Mapping**:
```python
# YNAB API response fields to use directly:
category_data = {
    "budgeted": response.data.category.budgeted_formatted,  # "£500.00"
    "activity": response.data.category.activity_formatted,   # "-£450.00"  
    "balance": response.data.category.balance_formatted,    # "£50.00"
}
```

**Error Handling**:
- Graceful fallback to raw milliunit fields if formatted fields are missing
- Clear logging for missing formatted fields
- Maintain system functionality even if formatting unavailable

**Testing Requirements**:
- Verify all schemas use formatted fields correctly
- Test with different currency types (£, $, €, etc.)
- Confirm fallback behavior works as expected

**Benefits**:
- **Zero Implementation**: Use existing YNAB functionality
- **Perfect Formatting**: YNAB handles all localization and formatting
- **Maintenance-Free**: No custom formatting code to maintain
- **Consistency**: Matches YNAB's official UI and API formatting
- **Reliability**: Battle-tested by YNAB across millions of users

### 8. Schema Enhancement Strategy

**Decision**: Enhance schemas to include activity-related data and collapse complex goal fields into human-readable summaries.

**Rationale**:
- Current schemas miss important budget fields (activity, balance, budgeted) that are available in YNAB API
- YNAB's goal system is complex with 10+ fields that are difficult for LLMs to interpret
- Human-readable goal summaries make the data more accessible to AI agents and end users
- Maintains all original technical information while providing better usability

**Implementation Details**:

**Activity Data Addition**:
- Add `activity`, `balance`, and `budgeted` fields to `CleanCategory` schema
- Add corresponding formatted currency fields (`activity_formatted`, `balance_formatted`, `budgeted_formatted`)
- Ensure all monetary fields include currency information
- Make new formatted fields required, keep milliunit fields for backward compatibility

**Goal Field Collapsing**:
- Create a goal transformation function that converts YNAB's complex goal fields into 2-3 human-readable strings
- Generate `goal_summary` with overall goal description and purpose
- Generate `goal_status` with current progress and any issues (underfunded, snoozed, etc.)
- Generate `goal_technical_details` with raw data for advanced use cases
- Handle all YNAB goal types (TB, TBD, MF, NEED) appropriately
- Include error handling for missing or inconsistent goal data

**Schema Changes Required**:
1. `CleanCategory` schema: Add activity-related fields and goal summary fields
2. `CategoryResponse` schema: Include new fields in single category responses
3. `CategoriesResponse` schema: Ensure compatibility with enhanced category data
4. Create goal transformation utility function in `schema_utils.py`

**Alternatives Considered**:
- Keep all YNAB goal fields separate (rejected due to complexity for LLMs)
- Create multiple specialized goal schemas (rejected as overly complex)
- Remove goal data entirely (rejected as losing valuable information)

## Risks / Trade-offs

### Risk: Overassignment Detection
**[Risk]** Agent might create negative Ready to Assign balances if not careful
**[Mitigation]** `check_budget_health` tool detects this condition and `reassign_money` validates sufficient funds before transfers

### Risk: Race Conditions
**[Risk]** Concurrent modifications could cause inconsistencies
**[Mitigation]** Atomic transaction-based approach and validation of current state before modifications

### Risk: Complexity Increase
**[Risk]** Adding write operations increases system complexity
**[Mitigation]** Specialized tools hide complexity from agent, maintain clear separation of concerns

### Risk: YNAB API Limitations
**[Risk]** YNAB API might have undocumented limitations on write operations
**[Mitigation]** Comprehensive testing with real YNAB data, graceful error handling

### Trade-off: Safety vs Flexibility
**[Trade-off]** Specialized tools are safer but less flexible than generic write access
**[Decision]** Prioritize safety for budgeting operations where errors can have financial consequences

## Migration Plan

### Deployment Steps:
1. Add new service methods to `ynab_service.py`
2. Create new schema files for request/response validation
3. Implement new tools in dedicated module
4. Register tools in server
5. Add comprehensive unit tests
6. Deploy to staging for testing
7. Deploy to production

### Rollback Strategy:
- New tools are additive (no breaking changes to existing functionality)
- Can be disabled by removing tool registration
- No database migrations required
- Simple configuration flag can disable write operations if needed

## Open Questions

1. Should we add rate limiting to write operations?
2. Do we need additional logging for audit purposes beyond what YNAB provides?
3. Should we implement a dry-run mode for testing?
4. What's the optimal error recovery strategy for failed transactions?
