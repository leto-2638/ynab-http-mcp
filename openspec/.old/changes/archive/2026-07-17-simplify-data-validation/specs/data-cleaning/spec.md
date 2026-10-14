## ADDED Requirements

### Requirement: Unified data cleaning function
The system SHALL provide a `clean_ynab_data()` function that performs all necessary data transformations in a single pass.

#### Scenario: Clean YNAB API response
- **WHEN** YNAB API response data is passed to `clean_ynab_data()`
- **THEN** the function SHALL return cleaned data with UUIDs converted to strings, import fields removed, and consistent data types

#### Scenario: Handle nested data structures
- **WHEN** nested YNAB data (transactions with subtransactions, categories with groups) is processed
- **THEN** the function SHALL recursively clean all levels of the data structure

#### Scenario: Preserve non-import fields
- **WHEN** data contains both import and non-import fields
- **THEN** the function SHALL remove only import-related fields and preserve all other fields

## MODIFIED Requirements

### Requirement: Data validation approach
The system SHALL use a simplified validation approach with single-pass validation instead of multiple validation layers.

#### Scenario: Validate cleaned data
- **WHEN** cleaned data is validated against Pydantic models
- **THEN** the system SHALL use basic Pydantic validation without custom error handling

#### Scenario: Handle validation errors
- **WHEN** data fails validation
- **THEN** the system SHALL raise standard Pydantic ValidationError with sufficient debugging information