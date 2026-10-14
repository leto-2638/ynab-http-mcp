## ADDED Requirements

### Requirement: Agent-friendly data types
All tool responses SHALL use data types that are easily consumable by AI agents.

#### Scenario: Use primitive types
- **WHEN** returning data to agents
- **THEN** the response SHALL contain only primitive JSON-serializable types

#### Scenario: Consistent string formats
- **WHEN** returning dates or identifiers
- **THEN** the system SHALL use ISO string formats (YYYY-MM-DD for dates, UUID strings for IDs)

## MODIFIED Requirements

### Requirement: Data type consistency
The system SHALL ensure consistent data types across all tool responses.

#### Scenario: Uniform data representation
- **WHEN** multiple tools return similar data types
- **THEN** the representation SHALL be consistent (e.g., all IDs as strings, all dates as ISO strings)

#### Scenario: Remove type conversion burden
- **WHEN** agents consume tool responses
- **THEN** agents SHALL not need to perform additional type conversions