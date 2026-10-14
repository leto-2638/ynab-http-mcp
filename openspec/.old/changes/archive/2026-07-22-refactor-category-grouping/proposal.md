## Why

The just-archived `refactor-category-schema` change flattened `data://categories` to a single `{"categories": [...]}` list, dropping the YNAB-native `category_groups` envelope. Real consumers (and the YNAB web UI) group categories by their parent group, and an LLM reasoning about budget structure benefits from that grouping being present in the response. We re-introduce a thin `MCPCategoryGroup` wrapper so the wire shape matches YNAB's natural model, while keeping the simplified per-category fields and `goal_summary`/`goal_status` strings from the previous change.

## What Changes

- **Add** `MCPCategoryGroup(MCPResponse[ynab.CategoryGroupWithCategories])`: a Pydantic model with `id`, `name`, `hidden`, `internal`, `deleted`, and `categories: List[MCPCategory]`, plus a `HIDE_DELETED: ClassVar[bool] = True` constant. A `from_ynab(raw)` classmethod that skips deleted groups/categories and nests `MCPCategory.from_ynab`.
- **Modify** `MCPCategories`: hold `category_groups: List[MCPCategoryGroup]` instead of the flat `categories` list. The wire shape becomes `{"category_groups": [{"id","name","categories": [...]}]}`. `HIDE_DELETED` stays `True`.
- **Keep** `MCPCategory` and `MCPCategoryGoal` unchanged: the single-category endpoint `data://categories/{category_id}` still returns a bare `MCPCategory` (no group envelope), and the goal-helper logic is preserved.
- **Update** `tests/test_categories_schema.py` to assert the new grouped shape (flat-list `HIDE_DELETED` tests are replaced with per-group and per-list deleted-filter tests).

**BREAKING**: the wire shape of `data://categories` flips from `{"categories": [...]}` to `{"category_groups": [{"id","name","categories": [...]}]}`. This is the second such flip in two changes; the schema files are versioned in archive history.

## Capabilities

### New Capabilities
- `category-grouping`: nested `MCPCategoryGroup` model holding `List[MCPCategory]`, with its own `HIDE_DELETED` filter, slotted into `MCPCategories` so the categories resource response matches YNAB's group-then-categories shape.

### Modified Capabilities
- `category-resource-schema`: the `MCPCategories` requirement changes from a flat list to a list of groups; the "no grouped wrapper" requirement is removed in favour of the new `MCPCategoryGroup` requirement.
- `ynab-mcp-resources`: the Categories Resource scenarios change to describe the grouped wire shape.

## Impact

- `src/ynab_http_mcp/schemas/categories.py` — add `MCPCategoryGroup`, rewrite `MCPCategories`.
- `src/ynab_http_mcp/schemas/__init__.py` — export `MCPCategoryGroup`.
- `tests/test_categories_schema.py` — replace flat-list tests with grouped tests.
- `tests/test_schema_metadata.py` — include `MCPCategoryGroup` in the registry.
- `tools/categories.py` — no code change (still calls `MCPCategories.from_ynab`).
- MCP wire shape for `data://categories` flips.
