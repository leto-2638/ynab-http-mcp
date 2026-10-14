## Context

The just-archived `refactor-category-schema` change (2026-07-22) delivered clean `MCPCategory` / `MCPCategoryGoal` / `MCPCategories` Pydantic models and rewired `tools/categories.py` to mirror `tools/accounts.py`. One outcome was a wire shape of `{"categories": [...]}` for `data://categories` — a flat list with no YNAB-native group structure. In real use (and the original broken schema), the YNAB SDK returns categories nested under `category_groups[*].categories[*]`. The flat shape strips out information the LLM and other consumers want.

We add an intermediate `MCPCategoryGroup` between `MCPCategory` and `MCPCategories`, restoring the grouped wire shape. The single-category endpoint stays bare (`MCPCategory`), the goal helpers stay untouched, and the schema layer still extends `MCPResponse[T]` exactly as before.

## Goals / Non-Goals

**Goals:**
- Restore the `category_groups[*].categories[*]` wire shape for `data://categories`.
- Add `MCPCategoryGroup(MCPResponse[ynab.CategoryGroupWithCategories])` with the same `HIDE_DELETED: ClassVar[bool] = True` filter pattern used elsewhere.
- Reuse `MCPCategory.from_ynab` so the goal-helper logic, formatted currency strings, and nested `goal` slot are preserved inside each group.
- Keep `MCPCategories` extending `MCPResponse[ynab.CategoriesResponse]` and accepting the same `HIDE_DELETED` override pattern.
- Keep the single-category endpoint (`data://categories/{category_id}`) returning a bare `MCPCategory` — no group envelope around it.

**Non-Goals:**
- Re-introducing a `CategoryGroup` *BaseModel* wrapper (the original broken code had `CategoryGroup(BaseModel)` outside the `MCPResponse` family). `MCPCategoryGroup` extends `MCPResponse[ynab.CategoryGroupWithCategories]` and uses the same `from_ynab` contract as every other resource model.
- Re-introducing `CategoriesResponse` / `CategoryResponse` — these were deleted for a reason and stay deleted.
- Changing the `MCPCategoryGoal` shape or the goal-helper logic.
- Changing the `tools/categories.py` structure (it still just calls `MCPCategories.from_ynab`).

## Decisions

### Decision 1: `MCPCategoryGroup` extends `MCPResponse[ynab.CategoryGroupWithCategories]`

Mirrors `MCPAccount` / `MCPAccounts` / `MCPCategories`: the model is a typed `MCPResponse[T]` and exposes a `from_ynab` classmethod. The YNAB SDK already returns `CategoryGroupWithCategories` (an extended `CategoryGroup` that carries the `categories` list) inside `CategoriesResponse.data.category_groups`, so we bind to that exact type.

**Alternatives considered:**
- Bind to bare `ynab.CategoryGroup` and read categories from a separate `data` slot. Rejected — the SDK's `CategoryGroupWithCategories` is the right type for the response data path; using anything else forces us to translate the grouping ourselves.

### Decision 2: `MCPCategories` holds `List[MCPCategoryGroup]`, not a flat list

`MCPCategories` is a higher-level wrapper around `ynab.CategoriesResponse`. Its `from_ynab` iterates `raw.data.category_groups`, skipping any group where `HIDE_DELETED and group.deleted`, then builds an `MCPCategoryGroup` per group. Each `MCPCategoryGroup.from_ynab` then iterates its own `categories`, skipping per-category deleted entries. The wire shape becomes `{"category_groups": [...]}`.

**Alternatives considered:**
- Keep the flat `{"categories": [...]}` and add a parallel `{"category_groups": [...]}`. Rejected — duplicates data and increases payload size; LLMs can derive groupings from `category_group_id` if they want a flat view.
- Nest categories inside groups but also expose a flat `categories` list. Rejected as YAGNI.

### Decision 3: `HIDE_DELETED` exists on both `MCPCategoryGroup` and `MCPCategories`

Both classes get `HIDE_DELETED: ClassVar[bool] = True`. The list-level filter skips deleted *groups*; the per-group filter skips deleted *categories*. Disabling both is possible by setting `HIDE_DELETED = False` on the appropriate subclass, and matches the pattern already in `MCPAccounts` and `MCPTransactions`.

**Alternatives considered:**
- Centralize the deleted-filter logic in `MCPCategories.from_ynab` only and pass an opt-in flag down. Rejected — `HIDE_DELETED` on the inner class lets consumers filter the inner list independently (e.g. when constructing an `MCPCategoryGroup` directly from a `ynab.CategoryGroupWithCategories`).

### Decision 4: Single-category endpoint stays bare `MCPCategory`

`tools/categories.py::get_category` calls `MCPCategory.from_ynab` on a raw `ynab.CategoryResponse`. Wrapping the single response in a group just to match the list shape would force the consumer to do `data.category.category_group_id` lookups for no benefit. The grouped shape belongs on the list endpoint; the single endpoint already gives the consumer every field they need.

**Alternatives considered:**
- Return `{"category": MCPCategory, "category_group": MCPCategoryGroup}` for the single endpoint. Rejected — the group is one indexed list, the LLM can pick the matching group by id if it cares.

## Risks / Trade-offs

- [Risk] Wire shape for `data://categories` flips for the second time in two changes. → Mitigation: archive history preserves the full spec; LLMs re-read the schema on the next request.
- [Risk] The grouped shape re-exposes the structure we just removed; if downstream consumers cache the flat shape they will break. → Mitigation: this is a personal MCP with no external consumers; single-user deployment noted in `openspec/config.yaml`.
- [Risk] Adding a third MCP model class makes the schemas package larger. → Mitigation: each class is small (≤ 50 LOC) and follows the same pattern; mypy and ruff keep the surface area typed and consistent.

## Migration Plan

1. Add `MCPCategoryGroup` to `schemas/categories.py`; update `MCPCategories` to hold `List[MCPCategoryGroup]`.
2. Update `schemas/__init__.py` to export `MCPCategoryGroup`.
3. Replace the flat-list unit tests in `tests/test_categories_schema.py` with grouped-shape tests.
4. Add `MCPCategoryGroup` to the registry expectations in `tests/test_schema_metadata.py`.
5. Run `uv run mypy src/ynab_http_mcp` and `uv run ruff check src/` — both must pass.
6. Rollback: `git revert` the single commit.

## Open Questions

- None.
