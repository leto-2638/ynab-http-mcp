## ADDED Requirements

### Requirement: Month-specific category lookup
The system SHALL provide an endpoint to retrieve category details for a specific month at `data://months/{month}/categories/{category_id}`.

#### Scenario: Retrieve category for specific month
- **WHEN** a client requests `data://months/2023-12/categories/abc123`
- **THEN** the system returns JSON data containing the category details for category abc123 in December 2023

#### Scenario: Handle non-existent category
- **WHEN** a client requests `data://months/2023-12/categories/nonexistent`
- **THEN** the system returns a 404 error with appropriate error message

### Requirement: Category data includes month context
The system SHALL include month-specific information in the category response.

#### Scenario: Month context in response
- **WHEN** examining the response from `data://months/2023-12/categories/abc123`
- **THEN** the response includes month-specific fields like budgeted amount for that month

### Requirement: Consistent error handling
The system SHALL handle errors consistently with other resource endpoints.

#### Scenario: Invalid month format
- **WHEN** a client requests `data://months/invalid-month/categories/abc123`
- **THEN** the system returns a 400 error with validation message
