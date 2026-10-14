## Why

This change adds budget management tools to enable an LLM-based budgeting advice agent to actively help users manage their YNAB budgets. Currently, the system only supports read-only operations. The new tools will allow the agent to reassign money between categories, check budget health, and provide actionable spending insights - transforming it from a passive data provider to an active budgeting assistant.

## What Changes

- Add three new MCP tools for budget management:
  - `reassign_money`: Move money between categories or from Ready to Assign
  - `check_budget_health`: Detect overassignment and negative balances
  - `get_budget_insights`: Provide comprehensive cross-month category analysis with trends and goal tracking

- Extend YnabService with write operations:
  - `update_month_category()` for direct budget adjustments
  
- Add request/response schemas for write operations
- Maintain all existing read-only functionality

## Capabilities

### New Capabilities
- `budget-management`: Core budget management operations including money reassignment and health checking
- `budget-insights`: Comprehensive cross-month category analysis with goal tracking
- `write-operations`: Foundation for all write capabilities to YNAB

### Modified Capabilities
- None (all existing capabilities remain unchanged)

## Schema Enhancements

### Currency Format Standardization

**Decision**: Leverage YNAB API's existing `_formatted` properties to provide human-readable currency values across all schemas, replacing raw milliunit fields.

**Current Approach** (Complex milliunits):
```json
{
  "budgeted": 500000,      // Raw milliunits
  "activity": -450000,     // Raw milliunits
  "balance": 50000         // Raw milliunits
}
```

**New Approach** (Use YNAB's formatted fields):
```json
{
  "budgeted": "£500.00",      // YNAB's budgeted_formatted
  "activity": "-£450.00",     // YNAB's activity_formatted  
  "balance": "£50.00"         // YNAB's balance_formatted
}
```

**Implementation Requirements**:
- **Use YNAB's existing `_formatted` properties directly** from API responses
- Replace all raw milliunit fields with their formatted equivalents
- Apply consistently across ALL schemas:
  - `CleanCategory`: Use `budgeted_formatted`, `balance_formatted`, `activity_formatted`
  - `MonthCategory`: Use `budgeted_formatted`, `activity_formatted`, `balance_formatted`
  - `Transaction`: Use `amount_formatted`
  - `Account`: Use `balance_formatted`
  - Any other schemas containing monetary values

**Key Insights**:
- YNAB API already provides properly formatted currency strings
- Formatted fields include correct currency symbols based on plan settings
- Values are properly rounded with appropriate decimal places
- Negative values correctly indicate spending/outflows
- No need to build our own formatting system

**Benefits**:
- **Zero Implementation Complexity**: Use existing YNAB functionality
- **Perfect Localization**: YNAB handles currency symbols and formatting
- **Consistency**: Uniform format matching YNAB's UI
- **Reliability**: Battle-tested formatting logic
- **Maintenance-Free**: Updates automatically with YNAB changes

### Goal Field Collapsing
The `CleanCategory` schema will collapse the numerous YNAB goal fields into 2-3 human-readable strings that provide comprehensive goal information in natural language format. This transformation addresses the complexity of YNAB's goal system while maintaining all necessary information for budgeting decisions.

**Current YNAB Goal Fields** (from [YNAB API documentation](https://api.ynab.com/v1#tag/categories/GET/plans/%7Bplan_id%7D/categories/%7Bcategory_id%7D)):
- `goal_type`: TB, TBD, MF, NEED (Target Balance, Target Balance by Date, Monthly Funding, Plan Your Spending)
- `goal_creation_month`: When the goal was created
- `goal_target`: Target amount in milliunits
- `goal_target_month`: Target month for completion
- `goal_percentage_complete`: Progress toward goal
- `goal_under_funded`: Amount needed this month to stay on track
- `goal_overall_funded`: Total amount funded so far
- `goal_overall_left`: Remaining amount needed
- `goal_months_to_budget`: Months remaining in goal period
- `goal_cadence`: Funding frequency
- `goal_cadence_frequency`: How often funding occurs

**Proposed Collapsed Format**:
```json
{
  "goal_summary": "This category has a monthly funding goal created in January 2024 to save £500 per month for a vacation. Target completion is December 2024.",
  "goal_status": "Currently £1,200 of £6,000 funded (20% complete). This month is underfunded by £150, but the goal is snoozed until June.",
  "goal_technical_details": "Type: MF, Target: £6000, Funded: £1200, Remaining: £4800, Months left: 7"
}
```

**Benefits**:
- **LLM-Friendly**: Natural language format is easier for AI agents to understand and explain to users
- **Comprehensive**: Maintains all original information in human-readable form
- **Contextual**: Provides both summary and detailed technical information
- **Flexible**: Adapts to different goal types (monthly funding, target balance, etc.)

### Activity Data Addition
The schema will be enhanced to include activity-related fields that are currently available in the YNAB API but not exposed in our simplified models:

**Fields to Add**:
- `activity`: Net activity/transactions for the category (in milliunits)
- `balance`: Current available balance (in milliunits)
- `budgeted`: Amount budgeted for the current month (in milliunits)

**Implementation Approach**:
- Add these fields to `CleanCategory` schema for single category views
- Ensure `MonthCategory` schema (used for monthly data) already includes these fields
- Maintain backward compatibility by making new fields optional where appropriate

## Impact

- **Code**: New files in `tools/`, extensions to `ynab_service.py`, schema enhancements
- **API**: New MCP tools added (no breaking changes to existing resources)
- **Dependencies**: No new external dependencies
- **Testing**: Unit tests needed for new tools, service methods, and schema transformations
