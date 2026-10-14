## Context

The server currently advertises eight explicit FastMCP tools plus the two generic tools added by `ResourcesAsTools`. An in-memory FastMCP client measured the compact `tools/list` payload at 12,436 characters. The native resource catalog contains four static resources and eighteen templates and measures 9,847 compact JSON characters.

The main tool cost is repeated prose inside nested Pydantic request-field descriptions, not function docstrings. The main resource cost is repeated Lean/Full explanation in drill-in resource descriptions. Claude and Hermes can defer MCP tools client-side, while other clients load definitions eagerly; the server therefore needs a smaller standard MCP surface without relying on client-specific behavior.

## Goals / Non-Goals

**Goals:**
- Reduce the serialized `tools/list` catalog to at most 10,000 compact JSON characters.
- Reduce the combined static-resource and resource-template catalog to at most 6,500 compact JSON characters.
- Preserve every tool name, input field, validation constraint, annotation, resource URI, response shape, and Full drill-in path.
- Keep enough prose for an agent to understand units, date formats, nullable behavior, side effects, and distinctions not encoded structurally.
- Prevent future catalog growth from going unnoticed.

**Non-Goals:**
- Implement a search/describe/call meta-protocol.
- Add vendor-specific deferred-loading or tool-search metadata.
- Hide or remove Full resources.
- Remove `ResourcesAsTools`.
- Change Lean, Full, or Aggregate response payloads.
- Change the user-global OpenCode or oh-my-opencode-slim configuration.

## Decisions

### 1. Optimize serialized metadata, not source line count

Descriptions will be shortened where they appear in the actual FastMCP wire catalog. For explicit tools, this primarily means Pydantic `Field(description=...)` values in request schemas. Replacing repeated Python strings with constants or deleting unregistered modules may improve maintainability but does not reduce the serialized catalog and is outside this change unless independently required by an edited path.

Machine-readable constraints remain authoritative: required fields, enums, numeric bounds, string lengths, and date patterns must not change. Prose will retain information JSON Schema cannot express clearly, especially milliunit semantics, inclusive/exclusive date boundaries, `null` behavior, and destructive effects.

**Alternative considered:** Trim only function docstrings. Rejected because top-level descriptions account for a small minority of the measured tool-description payload.

### 2. Keep Full resources discoverable, but describe them once and briefly

Every existing `/{id}/full` template remains registered and returned by standard resource discovery and `list_resources`. Each Full description will use a concise, entity-specific sentence that identifies it as a drill-in, names `full_details`, and says when it is appropriate, such as raw SDK-field access or milliunit arithmetic.

This preserves the progressive model:

```text
collection → lean entity → Full drill-in
                         ↘ related transactions / aggregates
```

**Alternative considered:** Hide Full templates by default. Rejected because the catalog saving is modest, tag filtering is client-dependent, and invisible Full paths weaken the documented escape hatch.

### 3. Retain the two-tool resource gateway

`ResourcesAsTools` remains enabled. It adds only `list_resources` and `read_resource`, rather than one tool per resource, and is required for clients such as Claude and Hermes that do not reliably expose native MCP resources in the agent loop.

**Alternative considered:** Remove the transform and rely on native `resources/list` and `resources/read`. Rejected because it would reduce cross-client accessibility to the read layer.

### 4. Use standard MCP behavior only

The server will not add a custom server-side search/describe/call bridge. Adding such a bridge while retaining normal tools would increase context; replacing typed tools with generic invocation would weaken validation, safety annotations, approvals, and call reliability.

The server will also avoid undocumented vendor `_meta` fields. Client implementations remain free to defer or search the standard catalog.

### 5. Enforce deterministic catalog budgets

Tests will instantiate an in-memory FastMCP `Client`, call `list_tools()`, `list_resources()`, and `list_resource_templates()`, serialize `model_dump(mode="json", exclude_none=True)` results using compact JSON separators, and assert the agreed character budgets.

Separate semantic-preservation assertions will verify:
- the registered tool and resource names remain present;
- required fields, enums, bounds, and standard MCP annotations remain unchanged;
- monetary inputs still identify milliunits;
- dates still identify accepted formats and boundary semantics where applicable;
- Full descriptions identify `full_details` and their drill-in purpose.

Character budgets are used instead of tokenizer-specific estimates so tests are deterministic across model providers.

## Risks / Trade-offs

- **[Risk] Over-trimming makes similar goal operations harder to distinguish.** → Preserve one concise operation-specific purpose and all semantics not encoded by JSON Schema; test representative descriptions.
- **[Risk] Hard size budgets may fail after a FastMCP serialization change.** → Treat a failure as a review gate: inspect the wire diff and deliberately update the budget only when the added metadata is valuable.
- **[Risk] Full resources remain visible in resource catalogs.** → Accept the small URI/name cost and remove repeated explanatory essays instead; discoverability is part of the Lean/Full contract.
- **[Trade-off] Vendor-specific search hints could improve one client.** → Prefer portable MCP output and avoid custom FastMCP serialization internals at the current catalog size.

## Migration Plan

1. Record the baseline catalog measurements in tests.
2. Trim explicit tool request-field descriptions without changing schemas or annotations.
3. Compress resource and Full-drill-in descriptions while preserving all registrations.
4. Run focused catalog, schema, and resource-registration tests, then `ruff` and `mypy` on affected code.
5. Restart the MCP service and verify `list_resources`, a Lean read, a Full read, and representative read/write tool discovery.

Rollback consists of restoring the previous description text and catalog budgets. No YNAB data migration, endpoint migration, or client configuration change is required.

## Open Questions

None. Client-side lazy loading remains an external optimization and can be adopted independently when supported by each client.
