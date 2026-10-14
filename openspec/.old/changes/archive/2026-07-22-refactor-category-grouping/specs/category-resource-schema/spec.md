## MODIFIED Requirements

### Requirement: MCPCategories is a flat-list wrapper with HIDE_DELETED

The system SHALL define `MCPCategories(MCPResponse[ynab.CategoriesResponse])` as a higher-level wrapper that holds `category_groups: List[MCPCategoryGroup]` and a class constant `HIDE_DELETED: bool = True`. The classmethod `from_ynab(raw: ynab.CategoriesResponse) -> MCPCategories` SHALL iterate `raw.data.category_groups`, skip groups where `HIDE_DELETED and group.deleted` is True, build each remaining group via `MCPCategoryGroup.from_ynab`, and return a `MCPCategories` containing the assembled list. The wire shape MUST match the YNAB-native `{"category_groups": [...]}` envelope (the flat-list shape from the previous change is removed).

#### Scenario: Default behaviour filters deleted groups and categories
- **WHEN** `MCPCategories.from_ynab` is called on a response containing a deleted group and a non-deleted group that itself contains a deleted category
- **THEN** the deleted group is omitted from the result
- **AND** the surviving group's `categories` list contains only non-deleted categories

#### Scenario: HIDE_DELETED can be disabled
- **WHEN** a subclass sets `HIDE_DELETED = False`
- **THEN** `from_ynab` includes deleted groups in the returned list
- **AND** each surviving group's `MCPCategoryGroup.from_ynab` still applies its own `HIDE_DELETED` filter on the inner categories list (settable independently on `MCPCategoryGroup`)

## REMOVED Requirements

### Requirement: No grouped response wrapper exists
**Reason**: The grouped wrapper is being re-introduced in this change to restore the YNAB-native wire shape.
**Migration**: Consumers that relied on the flat `{"categories": [...]}` shape must switch to `{"category_groups": [{"categories": [...]}]}`. The single-category endpoint `data://categories/{category_id}` is unchanged and still returns a bare `MCPCategory`.

## ADDED Requirements

### Requirement: MCPCategoryGroup nests categories with its own HIDE_DELETED

The system SHALL define `MCPCategoryGroup(MCPResponse[ynab.CategoryGroupWithCategories])` with the YNAB category-group fields (`id`, `name`, `hidden`, `internal`, `deleted`) and a `categories: List[MCPCategory]` slot. The class constant `HIDE_DELETED: bool = True` SHALL live on the class. The classmethod `from_ynab(raw: ynab.CategoryGroupWithCategories) -> MCPCategoryGroup` SHALL iterate `raw.categories`, skip entries where `HIDE_DELETED and category.deleted` is True, build each remaining category via `MCPCategory.from_ynab`, and return a `MCPCategoryGroup` whose `categories` slot holds the filtered list.

#### Scenario: from_ynab populates group metadata and nested categories
- **WHEN** `MCPCategoryGroup.from_ynab` is called with a `ynab.CategoryGroupWithCategories` containing two non-deleted categories
- **THEN** the returned instance has `id`, `name`, `hidden`, `internal`, `deleted` copied from the raw
- **AND** the `categories` list contains two `MCPCategory` instances built via `MCPCategory.from_ynab`

#### Scenario: Default behaviour filters deleted categories inside a group
- **WHEN** `MCPCategoryGroup.from_ynab` is called with a raw group containing both deleted and non-deleted categories
- **THEN** the returned `categories` list contains only the non-deleted categories

#### Scenario: HIDE_DELETED can be disabled on the inner class
- **WHEN** a subclass sets `MCPCategoryGroup.HIDE_DELETED = False`
- **THEN** `from_ynab` includes deleted categories in the returned `categories` list
