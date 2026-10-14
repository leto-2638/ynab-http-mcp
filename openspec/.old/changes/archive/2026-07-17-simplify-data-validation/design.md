## Context

The current system uses multiple validation passes, custom error handling, and complex Pydantic models to process YNAB API data. This creates unnecessary complexity for what should be simple data cleaning to make responses agent-friendly.

## Goals / Non-Goals

**Goals:**
- Simplify data processing pipeline to reduce complexity and improve maintainability
- Create unified data cleaning function to handle all transformation needs
- Use simpler validation approach suitable for agent consumption
- Maintain data quality while reducing overhead
- Improve performance by eliminating redundant validation passes

**Non-Goals:**
- Change the YNAB API integration itself
- Modify the MCP tool interfaces (keep same parameters and return types)
- Add new functionality beyond simplification
- Change error reporting for end users (keep same error behavior)

## Decisions

### 1. Unified Data Cleaning Function
**Decision**: Create `clean_ynab_data()` function that handles all data transformation in a single pass
**Rationale**: Centralizes all data cleaning logic (UUID→string, import field filtering, type conversion) in one place
**Alternatives Considered**: 
- Keep individual cleaning in each tool (rejected - duplicates code)
- Use Pydantic validators (rejected - adds complexity)

### 2. Simplified Validation Approach
**Decision**: Use single-pass validation with basic Pydantic models
**Rationale**: Reduces validation overhead while maintaining data quality
**Alternatives Considered**:
- Remove validation entirely (rejected - need basic data quality checks)
- Keep multi-layer validation (rejected - unnecessary complexity)

### 3. Basic Data Types Only
**Decision**: Use only simple data types (strings, ints, floats, dates) in schemas
**Rationale**: Better for agent consumption, avoids UUID/datetime complexity
**Alternatives Considered**:
- Keep UUID objects (rejected - agents struggle with complex types)
- Use custom types (rejected - adds unnecessary complexity)

### 4. Remove Custom Error Handling
**Decision**: Use Pydantic's built-in validation errors instead of custom `MCPValidationError`
**Rationale**: Simplifies code, Pydantic errors are sufficient for debugging
**Alternatives Considered**:
- Keep custom errors (rejected - overkill for simple validation)
- Add error translation layer (rejected - unnecessary complexity)

## Risks / Trade-offs

### Risk: Less granular error reporting
**Mitigation**: Pydantic validation errors still provide sufficient debugging information. Add debug logging if needed.

### Risk: Performance impact of single cleaning pass
**Mitigation**: Single pass is actually faster than multiple validation passes. Benchmark if concerns arise.

### Risk: Breaking changes to existing consumers
**Mitigation**: Keep same tool interfaces and response structures. Only internal implementation changes.

### Risk: Loss of validation coverage
**Mitigation**: Basic Pydantic validation covers 95% of needed checks. Add specific validators only if gaps found.

## Migration Plan

1. **Phase 1**: Create `clean_ynab_data()` function and simplified schemas
2. **Phase 2**: Update one tool (e.g., transactions) to use new approach
3. **Phase 3**: Test and verify simplified approach works correctly
4. **Phase 4**: Apply same pattern to remaining tools
5. **Phase 5**: Remove unused complex validation code and tests
6. **Phase 6**: Update documentation and examples

**Rollback Strategy**: Git commit after each tool conversion allows easy rollback to previous version if issues arise.

## Open Questions

- Should we keep any of the existing schema metadata for FastMCP compatibility?
- Do we need to maintain backward compatibility with any existing consumers?
- Should debug logging be enhanced to compensate for simpler error handling?