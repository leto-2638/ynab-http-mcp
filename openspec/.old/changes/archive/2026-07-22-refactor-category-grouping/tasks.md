## 1. Schema

- [x] 1.1 Add `MCPCategoryGroup(MCPResponse[ynab.CategoryGroupWithCategories])` to `schemas/categories.py` with id / name / hidden / internal / deleted / categories and `HIDE_DELETED: ClassVar[bool] = True`
- [x] 1.2 Implement `MCPCategoryGroup.from_ynab(raw)` that iterates `raw.categories`, skips `HIDE_DELETED and category.deleted`, and nests `MCPCategory.from_ynab`
- [x] 1.3 Change `MCPCategories` to hold `category_groups: List[MCPCategoryGroup]` (not a flat list) and update its `from_ynab` to iterate `raw.data.category_groups`, skip deleted groups, and call `MCPCategoryGroup.from_ynab` for each

## 2. Wire-up

- [x] 2.1 Update `schemas/__init__.py` to export `MCPCategoryGroup` alongside the existing three symbols
- [x] 2.2 Confirm `tools/categories.py` still works as-is (it calls `MCPCategories.from_ynab` whose return shape changes, so the JSON body changes but the call site does not)

## 3. Tests

- [x] 3.1 Update `tests/test_categories_schema.py`: replace the flat-list `MCPCategories` tests with grouped tests covering (a) default deleted-group + deleted-category filter, (b) HIDE_DELETED=False on `MCPCategories` and on `MCPCategoryGroup` independently, (c) nested `MCPCategory` shape inside each group
- [x] 3.2 Add a unit test for `MCPCategoryGroup.from_ynab` that builds a group with two categories and asserts id / name / hidden / internal / deleted are copied and the `categories` list is populated
- [x] 3.3 Update `tests/test_schema_metadata.py` to include `MCPCategoryGroup` in the public registry expectations
- [x] 3.4 Update `TestPublicSchemaSurface` to assert that `MCPCategoryGroup` IS exported (and the old `CategoryGroup` / `CategoriesResponse` / `CategoryResponse` are still NOT exported)

## 4. Verification

- [x] 4.1 Run `uv run mypy src/ynab_http_mcp` — must pass
- [x] 4.2 Run `uv run ruff check src/ynab_http_mcp/schemas/ src/ynab_http_mcp/tools/categories.py` — must pass
- [x] 4.3 Run the full test suite — new tests pass, no regressions in the 39 pre-existing categories-schema tests
- [x] 4.4 Boot the MCP server and confirm `data://categories` and `data://categories/{category_id}` still register (live API call requires a YNAB token we don't have; offline JSON shape can be verified via a quick pydantic round-trip)
