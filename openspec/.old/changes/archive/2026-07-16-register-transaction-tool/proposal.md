## Why

This change adds a transaction registration tool to the YNAB HTTP MCP server, enabling LLMs to register transactions programmatically. This capability is essential for automating transaction entry and providing insights based on real-time spending data.

## What Changes

- Add a new MCP tool endpoint for registering transactions
- Implement transaction validation and error handling
- Add support for transaction metadata (memo, payee, category)
- Create unit tests for the new functionality

## Capabilities

### New Capabilities
- `register-transaction`: Tool endpoint for registering new transactions in YNAB
- `transaction-validation`: Validation logic for transaction data
- `transaction-metadata`: Support for additional transaction metadata fields

### Modified Capabilities
- None (this is a new feature with no existing capabilities to modify)

## Impact

- Adds new HTTP endpoint to the MCP server
- Introduces new dependencies on YNAB transaction API
- Requires updates to server error handling and logging
- Unit tests will be added to ensure reliability
