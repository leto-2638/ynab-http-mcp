## ADDED Requirements

### Requirement: Simplified Pydantic models
The system SHALL use simplified Pydantic models with basic data types suitable for agent consumption.

#### Scenario: Use simple data types
- **WHEN** defining Pydantic models for tool responses
- **THEN** the models SHALL use only basic types (str, int, float, date, bool, List, Dict)

#### Scenario: Avoid complex types
- **WHEN** YNAB API returns complex types like UUID objects
- **THEN** the models SHALL convert them to simple string representations

## MODIFIED Requirements

### Requirement: Schema validation complexity
The system SHALL reduce schema validation complexity by removing unnecessary validation layers.

#### Scenario: Single validation pass
- **WHEN** processing tool responses
- **THEN** the system SHALL perform validation in a single pass instead of multiple layers

#### Scenario: Remove custom error handling
- **WHEN** validation fails
- **THEN** the system SHALL use Pydantic's built-in error handling instead of custom error classes