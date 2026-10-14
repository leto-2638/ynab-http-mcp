## 1. Catalog Measurement and Contract Tests

- [x] 1.1 Add a reusable test helper that opens the in-memory FastMCP server, lists tools/resources/templates, and serializes their model dumps with deterministic compact JSON.
- [x] 1.2 Add tool-catalog contract assertions for the complete registered name set, input argument shapes, required fields, constraints, enums, and standard MCP annotations.
- [x] 1.3 Add resource-catalog contract assertions for the complete URI/template set, including every Lean, Full, Aggregate, and related-transaction path.
- [x] 1.4 Add failing footprint assertions for the 10,000-character tool budget and 6,500-character combined resource/template budget.
- [x] 1.5 Add semantic-description assertions covering milliunits, date formats and boundaries, operation distinctions, Lean-first guidance, and `full_details` drill-in guidance.

## 2. Tool Schema Compression

- [x] 2.1 Shorten nested Pydantic request-field descriptions for category assignment, goal management, category details, goal clearing, transaction creation, budget health, and spending insights.
- [x] 2.2 Preserve every existing tool name, request wrapper, field name, required/optional state, enum, bound, format, and standard MCP annotation while trimming prose.
- [x] 2.3 Shorten top-level tool descriptions only where they contain redundant text, keeping each operation's purpose distinct.
- [x] 2.4 Run the focused tool-catalog tests and confirm the serialized tool definitions are at most 10,000 characters.

## 3. Resource Catalog Compression

- [x] 3.1 Replace repeated Full-resource convention essays with concise entity-specific drill-in descriptions that mention `full_details` and when Full is appropriate.
- [x] 3.2 Trim repeated transaction-filter examples and other verbose resource descriptions while preserving required filters, accepted formats, and inclusive/exclusive boundary semantics.
- [x] 3.3 Keep `ResourcesAsTools`, every existing resource/template registration, and the Lean → Full → Aggregate discovery model unchanged.
- [x] 3.4 Run the focused resource-catalog tests and confirm the combined static-resource and template definitions are at most 6,500 characters.

## 4. Verification

- [x] 4.1 Run the focused catalog-footprint, tool-schema, resource-registration, Lean/Full, and integration tests affected by the metadata changes.
- [x] 4.2 Run Ruff formatting/checks and mypy for the affected Python modules and tests.
- [x] 4.3 Check service status, restart the YNAB MCP server after the code changes, and verify service health.
- [x] 4.4 Smoke-test `list_resources`, one Lean resource, its Full counterpart, and representative read-only and destructive tool discovery without making an unintended YNAB mutation.
- [x] 4.5 Record the final catalog character counts and confirm no tool arguments, resource URIs, response shapes, or standard annotations changed.
