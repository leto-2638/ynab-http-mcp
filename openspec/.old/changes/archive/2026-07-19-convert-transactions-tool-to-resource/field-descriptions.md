# Official YNAB Transaction Field Descriptions

Based on official YNAB API documentation research:

## Transfer-Related Fields

### `transfer_account_id`
- **Type**: UUID (string)
- **Context**: Used in **subtransactions** to specify the target account for a transfer transaction
- **Description**: Identifies the account to which funds are being transferred. Only applicable in split transactions where one subtransaction represents a transfer from one account to another.
- **Example**: `"transfer_account_id": "550e8400-e29b-41d4-a716-446655440000"`

### `transfer_transaction_id`
- **Type**: UUID (string)
- **Context**: Used in **subtransactions** to link a transfer transaction to its counterpart
- **Description**: References the ID of the transaction that represents the reverse transfer (e.g., the debit from the source account when this is the credit to the destination account).
- **Example**: `"transfer_transaction_id": "330e8400-e29b-41d4-a716-446655440001"`

## Import and Matching Fields

### `matched_transaction_id`
- **Type**: UUID (string)
- **Context**: Used for **imported transactions**
- **Description**: Links a transaction to its matched counterpart in a reconciliation process. Used when imported transactions have been matched to existing transactions in YNAB to avoid duplicates.
- **Example**: `"matched_transaction_id": "550e8400-e29b-41d4-a716-446655440001"`

## Flagging System

### `flag_color`
- **Type**: enum (string)
- **Possible Values**: `red`, `orange`, `yellow`, `green`, `blue`, `purple`, `null`
- **Description**: Color code used to visually categorize transactions for prioritization or filtering.
- **Example**: `"flag_color": "red"`

### `flag_name`
- **Type**: string
- **Description**: Custom name for the transaction flag, allowing users to label transactions with specific identifiers.
- **Example**: `"flag_name": "Bill Due"`

## Debt Transactions

### `debt_transaction_type`
- **Type**: enum (string)
- **Possible Values**:
  - `loan_payment`
  - `loan_principal`
  - `loan_interest`
  - `credit_card_payment`
  - `credit_card_principal`
  - `credit_card_fee`
  - `null` (not a debt transaction)
- **Description**: Categorizes transactions related to debt accounts such as loans or credit cards.
- **Example**: `"debt_transaction_type": "loan_payment"`

## Core Transaction Fields

### `amount`
- **Type**: integer (int64)
- **Description**: Transaction amount in **milliunits** (1/1000 of the currency unit). For example, $1.00 = 1000 milliunits, $50.00 = 50000 milliunits.
- **Example**: `"amount": 10000` (represents $10.00)

### `cleared`
- **Type**: enum (string)
- **Possible Values and Meanings**:
  - `uncleared`: Transaction has not yet been matched to a bank statement
  - `cleared`: Transaction has been matched to a bank statement and is confirmed
  - `reconciled`: Transaction has been reconciled as part of a reconciliation process
- **Description**: Indicates the reconciliation status of the transaction.
- **Example**: `"cleared": "cleared"`

## Additional Important Context

### Subtransactions
- Used for **split transactions** where a single payment is allocated to multiple categories
- Each subtransaction can include transfer-related fields
- Example: A $100 grocery transaction split between "Groceries" ($80) and "Household" ($20)

### Imported Transactions
- Use `import_id` to uniquely identify transactions during import
- `matched_transaction_id` links imported transactions to existing records
- Helps with reconciliation and duplicate detection

### Transfer Transactions
- Represented using subtransactions with `transfer_account_id` and `transfer_transaction_id`
- One subtransaction shows the debit, another shows the credit
- Example: Transfer $100 from Checking to Savings creates two linked subtransactions