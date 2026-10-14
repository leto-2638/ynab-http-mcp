## ADDED Requirements

### Requirement: MCPCategory inherits from MCPResponse

The system SHALL define `MCPCategory(MCPResponse[ynab.Category])` with the same shape used by `MCPAccount`: a single class with all category fields, a classmethod `from_ynab(raw: ynab.Category | ynab.CategoryResponse) -> MCPCategory` that accepts either a bare `ynab.Category` or a wrapped `ynab.CategoryResponse` (unwrapping via `raw.data.category` in the latter case). No separate response wrapper for the single-category case.

#### Scenario: from_ynab accepts a bare ynab.Category
- **WHEN** `MCPCategory.from_ynab` is called with a `ynab.Category` instance
- **THEN** it returns a populated `MCPCategory` with id, name, group id, hidden, deleted, note, formatted budget/activity/balance, and a nested `MCPCategoryGoal` populated from the same raw object

#### Scenario: from_ynab accepts a wrapped ynab.CategoryResponse
- **WHEN** `MCPCategory.from_ynab` is called with a `ynab.CategoryResponse` instance
- **THEN** it unwraps `raw.data.category` and returns the same shape as the bare case

#### Scenario: MCPCategory surfaces formatted currency
- **WHEN** `MCPCategory.from_ynab` populates the budget fields
- **THEN** `budgeted_formatted`, `activity_formatted`, and `balance_formatted` contain YNAB's pre-formatted strings (e.g. `"-£450.00"`)
- **AND** no milliunit-to-decimal conversion is performed in this layer

### Requirement: MCPCategoryGoal produces LLM-friendly summary strings

The system SHALL define `MCPCategoryGoal(MCPResponse[ynab.Category])` with the full set of typed YNAB goal fields plus two new optional strings: `goal_summary` and `goal_status`. The `from_ynab` classmethod SHALL populate both by calling two module-level helpers — `_explain_goal_type(raw)` and `_explain_goal_funding_status(raw)` — and SHALL return an empty-instance `MCPCategoryGoal()` (all fields None) when `raw.goal_type` is None.

#### Scenario: No goal yields empty goal model
- **WHEN** `MCPCategoryGoal.from_ynab` is called with a `ynab.Category` whose `goal_type` is None
- **THEN** the returned instance has every field set to None, including `goal_summary` and `goal_status`

#### Scenario: TB goal explains target balance
- **WHEN** the raw category has `goal_type = "TB"` and a `goal_target_formatted` value
- **THEN** `goal_summary` contains the literal phrase "Target Category Balance" and references the formatted target amount
- **AND** `goal_status` reflects whether the category is currently under- or over-funded

#### Scenario: TBD goal explains target balance by date
- **WHEN** the raw category has `goal_type = "TBD"` and a `goal_target_date`
- **THEN** `goal_summary` contains "Target Category Balance by Date" and references the target date and formatted target amount

#### Scenario: MF goal explains cadence and refill vs set-aside
- **WHEN** the raw category has `goal_type = "MF"`
- **THEN** `goal_summary` describes the cadence in plain English (e.g. "every 2 months", "weekly", "every 2 years", "by <date>" when cadence is 0)
- **AND** the phrasing uses "Refill" when `goal_needs_whole_amount` is False and "Set aside" when True
- **AND** `_explain_goal_type` raises `ValueError` for any cadence value outside 0–14 or any missing required `goal_cadence_frequency`

#### Scenario: NEED goal explains plan-your-spending
- **WHEN** the raw category has `goal_type = "NEED"`
- **THEN** `goal_summary` contains "Plan Your Spending"

#### Scenario: Unknown goal type is labelled but not an error
- **WHEN** the raw category has an unrecognised `goal_type`
- **THEN** `goal_summary` returns `"Unknown goal type: {raw.goal_type}"` and `from_ynab` does not raise

### Requirement: MCPCategories is a flat-list wrapper with HIDE_DELETED

The system SHALL define `MCPCategories(MCPResponse[ynab.CategoriesResponse])` as a higher-level wrapper that holds a literal `List[MCPCategory]` and a class constant `HIDE_DELETED: bool = True`. The classmethod `from_ynab(raw: ynab.CategoriesResponse) -> MCPCategories` SHALL iterate `raw.data.category_groups[*].categories[*]`, skip categories where `HIDE_DELETED and category.deleted` is True, build each `MCPCategory` via `MCPCategory.from_ynab`, and return a `MCPCategories` containing the remaining list.

#### Scenario: Default behaviour filters deleted categories
- **WHEN** `MCPCategories.from_ynab` is called on a response containing both deleted and non-deleted categories
- **THEN** the returned list contains only non-deleted categories

#### Scenario: HIDE_DELETED can be disabled
- **WHEN** a subclass sets `HIDE_DELETED = False`
- **THEN** `from_ynab` includes deleted categories in the returned list

### Requirement: No grouped response wrapper exists

The system SHALL NOT define `CategoriesResponse`, `CategoryResponse`, `MCPCategoryWithGroup`, `MCPCategoryGroup`, or any equivalent grouped wrapper. The single-category resource returns a bare `MCPCategory` and the list resource returns a bare `MCPCategories` (a flat list), mirroring `data://accounts` and `data://accounts/{account_id}`.

#### Scenario: Old grouped wrappers are removed
- **WHEN** `schemas/categories.py` is imported
- **THEN** no symbol named `CategoriesResponse`, `CategoryResponse`, or `CategoryGroup` is exported
- **AND** the public schema exports are exactly `MCPCategory`, `MCPCategoryGoal`, and `MCPCategories`

### Requirement: Categories schema module imports cleanly under mypy and ruff

The system SHALL keep `src/ynab_http_mcp/schemas/categories.py` free of undefined names, broken indentation, stray braces, and unused imports. `uv run mypy src` and `uv run ruff check .` SHALL both succeed on the file.

#### Scenario: No undefined names remain
- **WHEN** `schemas/categories.py` is imported
- **THEN** no `NameError` is raised
- **AND** every type referenced in field annotations is imported
