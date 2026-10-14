## Why

Agents that eagerly load this server's MCP catalogs spend unnecessary context on repeated parameter prose and verbose resource descriptions. The server should provide a compact, portable surface while preserving typed safety, cross-client compatibility, and the Lean → Full drill-in path.

## What Changes

- Trim serialized tool and parameter descriptions, retaining constraints, units, date formats, side effects, and distinctions that affect correct calls.
- Replace repeated Full-resource essays with concise drill-in descriptions while keeping every Full URI discoverable.
- Keep `ResourcesAsTools` so agents without native MCP resource support can discover and read resources through two generic tools.
- Add automated catalog-footprint checks for `tools/list` and the resource/template catalog to prevent context-size regressions.
- Document measured catalog budgets and the Lean-first, Full-on-demand guidance used in resource descriptions.
- Do not add a server-side search/describe/call bridge or vendor-specific deferred-loading metadata; capable clients may apply their own tool search.

## Capabilities

### New Capabilities
- `mcp-context-efficiency`: Compact, semantically sufficient MCP tool and resource catalogs with measurable size limits and progressive Lean/Full guidance.

### Modified Capabilities

None. Existing resource URIs, tool names, input fields, response shapes, and Full-resource discoverability remain unchanged.

## Impact

- **Code**: FastMCP tool/resource descriptions and Pydantic request-field metadata in `src/ynab_http_mcp/tools/` and `src/ynab_http_mcp/schemas/`.
- **Tests**: Catalog serialization and semantic-preservation tests, including explicit size budgets.
- **API compatibility**: No tool, parameter, resource URI, or response-shape removals; descriptive metadata becomes shorter.
- **Clients**: Claude, Hermes, and clients that eagerly inject MCP definitions receive the same capabilities with less context overhead.
- **Dependencies**: No new runtime dependencies.
