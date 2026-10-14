## Context

The current codebase has inconsistent patterns between modules. The `categories.py` module uses `@mcp.resource()` decorators with `data://` URIs, while `planning.py` uses `@mcp.tool()` decorators. This creates an uneven API surface that's harder to maintain and extend.

## Goals / Non-Goals

**Goals:**
- Convert existing planning tools to MCP resources for consistency
- Add new month-category lookup functionality
- Maintain backward compatibility where possible
- Follow established patterns from categories module

**Non-Goals:**
- Changing the underlying YNAB API integration
- Modifying existing data schemas unnecessarily
- Breaking existing client integrations without migration path

## Decisions

### Resource Pattern Adoption
**Decision**: Convert all planning endpoints from `@mcp.tool()` to `@mcp.resource()` decorators
**Rationale**: 
- Creates consistency with categories module
- Resources are better suited for read-only data access patterns
- Enables better URI-based resource discovery
- Aligns with MCP framework best practices

### URI Structure
**Decision**: Use `data://months/{month_id}` pattern for month resources
**Rationale**: 
- Follows the same pattern as `data://categories/{category_id}`
- Clear hierarchical structure
- Enables future extension with sub-resources

### New Category Endpoint
**Decision**: Add `data://months/{month}/categories/{category_id}` endpoint
**Rationale**: 
- Provides direct access to category details within planning context
- Enables efficient data retrieval for specific use cases
- Follows RESTful resource hierarchy patterns

### Service Layer Extension
**Decision**: Add new methods to YnabService class instead of modifying existing ones
**Rationale**: 
- Minimizes risk to existing functionality
- Follows single responsibility principle
- Makes it easier to test new functionality independently

## Risks / Trade-offs

**[Risk] API Migration Complexity**: Existing clients using tool endpoints will need updates
→ **Mitigation**: Document migration path clearly, consider temporary backward compatibility layer if needed

**[Risk] Performance Impact**: New category lookup might add database queries
→ **Mitigation**: Implement efficient caching at service layer, monitor performance in testing

**[Risk] Schema Changes**: New response types might require schema updates
→ **Mitigation**: Reuse existing schemas where possible, only create new ones for genuinely new data structures

**[Trade-off] Consistency vs Backward Compatibility**: Full conversion breaks existing integrations
→ **Decision**: Prioritize consistency but document migration path clearly

## Migration Plan

1. **Phase 1 - Implementation**: 
   - Add new resource endpoints alongside existing tools
   - Implement new YnabService methods
   - Update schemas as needed

2. **Phase 2 - Testing**:
   - Verify all endpoints return correct data
   - Test error handling and edge cases
   - Performance testing for new category endpoint

3. **Phase 3 - Documentation**:
   - Update API documentation
   - Create migration guide for existing clients
   - Add examples for new endpoints

4. **Phase 4 - Deployment**:
   - Deploy with both old and new endpoints active
   - Monitor usage patterns
   - Deprecate old tool endpoints after migration period

## Open Questions

- Should we implement a temporary backward compatibility layer that routes old tool calls to new resources?
- What's the expected query volume for the new category endpoint? (affects caching strategy)
- Are there any existing clients that would be immediately broken by this change?
