## MODIFIED Requirements

### Requirement: Simplified data models
Existing data models SHALL be simplified to use basic data types and single validation pass.

#### Scenario: Convert existing models
- **WHEN** updating existing Pydantic models
- **THEN** the models SHALL be simplified to use basic types and remove complex configurations

#### Scenario: Maintain model compatibility
- **WHEN** simplifying models
- **THEN** the simplified models SHALL still validate the same data structures

## REMOVED Requirements

### Requirement: Complex schema metadata
**Reason**: Schema metadata generation is unnecessary overhead for simplified approach
**Migration**: Remove schema metadata generation code and related tests