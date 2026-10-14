## ADDED Requirements

### Requirement: Transactions Resource Endpoint
The system SHALL provide a resource endpoint for accessing transaction data with filtering capabilities.

#### Scenario: Resource endpoint exists
- **WHEN** MCP client requests `data://transactions`
- **THEN** the system SHALL return a valid resource response

#### Scenario: Resource supports query parameters
- **WHEN** MCP client requests `data://transactions?since_date=2024-01-01&type=cleared`
- **THEN** the system SHALL process the query parameters and return filtered transactions

### Requirement: Mandatory Filter Validation
The system SHALL require at least one of the core filters (`type`, `since_date`, or `until_date`) to be provided.

#### Scenario: Missing mandatory filters
- **WHEN** MCP client requests `data://transactions` with no filters
- **THEN** the system SHALL return an error response with message indicating missing mandatory filters

#### Scenario: Valid filter combination
- **WHEN** MCP client requests `data://transactions?type=cleared`
- **THEN** the system SHALL process the request successfully

#### Scenario: Multiple valid filters
- **WHEN** MCP client requests `data://transactions?since_date=2024-01-01&until_date=2024-01-31&type=cleared`
- **THEN** the system SHALL process the request successfully using all provided filters

### Requirement: Date Parameter Handling
The system SHALL accept date parameters in ISO format (YYYY-MM-DD) and convert them to datetime objects.

#### Scenario: Valid date format
- **WHEN** MCP client provides `since_date=2024-01-15`
- **THEN** the system SHALL convert it to a datetime object and use it for filtering

#### Scenario: Invalid date format
- **WHEN** MCP client provides `since_date=invalid-date`
- **THEN** the system SHALL return an error response with message indicating invalid date format

### Requirement: JSON Response Format
The system SHALL return transaction data as a JSON string with the same structure as the current tool response, enhanced with contextual hints.

#### Scenario: Successful response format
- **WHEN** MCP client requests valid transaction data
- **THEN** the system SHALL return a JSON string containing `transactions` array, `server_knowledge` field, and `_hints` metadata

#### Scenario: Hints provide field explanations
- **WHEN** MCP client requests transaction data
- **THEN** the system SHALL include a `_hints` object explaining complex fields like `transfer_account_id`, `matched_transaction_id`, etc.

#### Scenario: Error response format
- **WHEN** MCP client provides invalid parameters
- **THEN** the system SHALL return a JSON string containing `error` field with descriptive message

### Requirement: Filter Priority Logic
The system SHALL maintain the existing filter priority logic where account_id takes precedence over month, payee_id, and category_id.

#### Scenario: Account filter priority
- **WHEN** MCP client provides both `account_id` and `month` parameters
- **THEN** the system SHALL use only the `account_id` filter (as per existing logic)

#### Scenario: Month filter when no account
- **WHEN** MCP client provides `month` but no `account_id`
- **THEN** the system SHALL use the `month` filter for transactions

### Requirement: Contextual Field Hints
The system SHALL provide contextual hints and explanations for complex transaction fields to help agents understand the data.

#### Scenario: Hints for transfer-related fields
- **WHEN** MCP client requests transaction data
- **THEN** the system SHALL include hints explaining `transfer_account_id` and `transfer_transaction_id` fields, clarifying they are used in subtransactions for transfer transactions

#### Scenario: Hints for matched transactions
- **WHEN** MCP client requests transaction data
- **THEN** the system SHALL include hints explaining `matched_transaction_id` field is used for imported transactions to link to existing records

#### Scenario: Hints for flag and debt fields
- **WHEN** MCP client requests transaction data
- **THEN** the system SHALL include hints explaining `flag_color` (with all possible values), `flag_name`, and `debt_transaction_type` (with all possible values) fields

#### Scenario: Hints for amount and cleared fields
- **WHEN** MCP client requests transaction data
- **THEN** the system SHALL include hints explaining `amount` is in milliunits and `cleared` field with all possible values and their meanings

### Requirement: Data Cleaning and Validation
The system SHALL maintain the existing data cleaning and validation logic for transaction data.

#### Scenario: Invalid transaction data
- **WHEN** YNAB API returns transaction data that fails validation
- **THEN** the system SHALL skip invalid transactions but continue processing valid ones

#### Scenario: Valid transaction data
- **WHEN** YNAB API returns valid transaction data
- **THEN** the system SHALL clean and validate all transactions according to existing logic