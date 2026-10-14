## Context

The current transactions functionality is implemented as an MCP tool (`@mcp.tool()`) in the `transactions.py` module. This provides flexible filtering capabilities but doesn't follow the resource pattern established by other data endpoints like categories and accounts. The goal is to convert this to a resource-based interface while maintaining all existing functionality and adding the requirement that at least one of the core filters (`type`, `since_date`, or `until_date`) must be provided.

## Goals / Non-Goals

**Goals:**
- Convert the transactions tool to a resource using `@mcp.resource()` decorator
- Maintain all existing filtering capabilities (since_date, until_date, type, account_id, month, payee_id, category_id, limit)
- Implement mandatory filter validation (at least one of type, since_date, or until_date must be provided)
- Follow the same architectural pattern as existing resources (categories, accounts)
- Return JSON string responses instead of Pydantic models
- Provide clear error messages for validation failures

**Non-Goals:**
- Change the underlying data processing logic
- Modify the YNAB service layer
- Add new filtering capabilities beyond what currently exists
- Change the response data structure

## Decisions

### 1. Resource Template with Query Parameters
**Decision**: Use `data://transactions{?since_date,until_date,type,account_id,month,payee_id,category_id,limit}` URI template
**Rationale**: This follows FastMCP best practices for parameterized resources and allows flexible filtering via query parameters. It's consistent with RESTful API design patterns and provides a clean interface for clients.
**Alternatives Considered**: 
- Path parameters (e.g., `data://transactions/{type}`) - rejected because it would require multiple resource definitions for different filter combinations
- No parameters with POST body - rejected because resources should be read-only and GET-based

### 2. Mandatory Filter Validation
**Decision**: Implement validation logic that requires at least one of `type`, `since_date`, or `until_date` to be provided
**Rationale**: This ensures clients provide meaningful filtering criteria and prevents overly broad queries that could return excessive data.
**Implementation**: Add validation at the beginning of the resource function that checks for the presence of these parameters and returns a clear error message if none are provided.

### 3. JSON String Return Type with Contextual Hints
**Decision**: Return `json.dumps(response_dict)` with added `_hints` metadata field instead of Pydantic model
**Rationale**: This is consistent with existing resource implementations (categories, accounts) and provides better compatibility with MCP resource consumers that expect string content. The added hints help agents understand complex transaction fields.
**Alternatives Considered**: 
- Return Pydantic model directly - rejected because resources expect string/bytes content
- Use ResourceResult wrapper - rejected as unnecessary complexity for this use case
- Separate hints resource - rejected because inline hints provide immediate context

### 4. Contextual Hints Implementation
**Decision**: Add a `_hints` field to the response containing explanations for complex fields
**Rationale**: Helps AI agents understand YNAB-specific concepts like transfer transactions, matched transactions, and flagging system.
**Implementation**: 
- Add detailed descriptions to Pydantic schema fields
- Include a `_hints` object in the JSON response with key field explanations
- Focus on fields that are YNAB-specific or not self-explanatory
**Fields to Explain**:
- `transfer_account_id`: Used in subtransactions to specify target account for transfers
- `transfer_transaction_id`: Used in subtransactions to link to reverse transfer transaction
- `matched_transaction_id`: Used for imported transactions to link to existing records
- `flag_color`: Flag color enum with all possible values (red, orange, yellow, green, blue, purple)
- `flag_name`: Custom label for flagged transactions
- `debt_transaction_type`: Debt transaction enum with all possible values (loan_payment, loan_principal, loan_interest, credit_card_payment, credit_card_principal, credit_card_fee)
- `amount`: Transaction amount in milliunits (1/1000 of currency unit)
- `cleared`: Clearing status enum with values and meanings (uncleared, cleared, reconciled)

### 4. Error Handling Strategy
**Decision**: Return JSON-formatted error responses with clear messages for validation failures
**Rationale**: Provides better client experience and debugging capabilities. Consistent with REST API best practices.
**Implementation**: Use `json.dumps({"error": "message"})` for validation errors, maintaining the same JSON response format.

### 5. Parameter Conversion Strategy
**Decision**: Convert string parameters to appropriate types (datetime for dates) early in the function
**Rationale**: Maintains compatibility with the existing YNAB service layer and provides clear error messages for invalid date formats.
**Implementation**: Use `datetime.fromisoformat()` with try/catch for date parameters, returning validation errors for malformed dates.

## Risks / Trade-offs

### Risk: Breaking Change for Existing Clients
**Mitigation**: This is acknowledged as a breaking change in the proposal. The new resource interface provides equivalent functionality but with a different access pattern. Documentation will need to clearly explain the migration path.

### Risk: Performance Impact of Validation Logic
**Mitigation**: The validation logic is minimal (simple parameter presence checks) and should have negligible performance impact. The underlying data processing remains unchanged.

### Risk: Complex Query Parameter Combinations
**Mitigation**: The existing filtering logic in the YNAB service layer already handles complex parameter combinations. The resource template simply passes these through to the service layer.

### Risk: Date Format Validation Errors
**Mitigation**: Implement robust error handling for date parsing with clear error messages. The FastMCP framework will automatically handle type conversion for query parameters.

## Migration Plan

1. **Implementation**: Convert the tool to a resource while keeping the old tool temporarily
2. **Testing**: Verify the new resource works correctly with various filter combinations
3. **Documentation**: Update README and examples to show the new resource interface
4. **Deprecation**: Mark the old tool as deprecated (if keeping it temporarily)
5. **Removal**: Remove the old tool in a future version once clients have migrated

## Open Questions

- Should we keep the old tool temporarily for backward compatibility, or remove it immediately?
- What should the exact error message be for missing mandatory filters?
- Should we add rate limiting or other protections for the resource endpoint?