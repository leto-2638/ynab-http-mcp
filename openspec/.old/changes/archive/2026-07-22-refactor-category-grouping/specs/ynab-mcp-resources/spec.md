## MODIFIED Requirements

### Requirement: Categories Resource

The system SHALL provide an MCP resource for accessing category data.

#### Scenario: Categories resource available
- **WHEN** the categories resource is registered
- **THEN** a resource with URI `data://categories` is available
- **AND** the resource returns JSON data with MIME type `application/json`
- **AND** the response contains a `category_groups` array matching the `MCPCategories` grouped shape

#### Scenario: Categories resource data structure
- **WHEN** the categories resource is accessed
- **THEN** it returns data matching the `MCPCategories` schema
- **AND** the top-level `category_groups` array contains one `MCPCategoryGroup` per non-deleted group (filtered via `MCPCategories.HIDE_DELETED`)
- **AND** each group exposes a nested `categories` array of `MCPCategory` objects, each with formatted currency fields (`budgeted_formatted`, `activity_formatted`, `balance_formatted`) and, when a goal exists, a nested `MCPCategoryGoal` with `goal_summary` and `goal_status` plain-English strings

  ```json
  {
    "category_groups": [
      {
        "id": "string",
        "name": "string",
        "hidden": "boolean",
        "internal": "boolean",
        "deleted": "boolean",
        "categories": [
          {
            "id": "string",
            "category_group_id": "string",
            "name": "string",
            "hidden": "boolean",
            "internal": "boolean",
            "deleted": "boolean",
            "budgeted_formatted": "string",
            "activity_formatted": "string",
            "balance_formatted": "string",
            "goal": {
              "goal_type": "string",
              "goal_summary": "string",
              "goal_status": "string"
            }
          }
        ]
      }
    ]
  }
  ```

#### Scenario: Single-category resource returns a bare category
- **WHEN** the resource at URI `data://categories/{category_id}` is accessed
- **THEN** the response JSON contains a top-level `MCPCategory` (no `category_group` envelope, no `category_groups` array)
- **AND** the single-category endpoint is unchanged by the grouped refactor
