## MODIFIED Requirements

### Requirement: MCPCategoryGoal produces LLM-friendly summary strings

The system SHALL define `MCPCategoryGoal(MCPResponse[ynab.Category])` with the following fields: three raw fields — `goal_type` (string, nullable), `goal_target_date` (ISO date, nullable), `goal_percentage_complete` (integer 0..100, nullable) — plus two derived optional strings, `goal_summary` and `goal_status`. The `from_ynab` classmethod SHALL populate the two derived strings by calling two module-level helpers — `_explain_goal_type(raw)` and `_explain_goal_funding_status(raw)` — and SHALL return an empty-instance `MCPCategoryGoal()` (every field None) when `raw.goal_type` is None. The model SHALL NOT expose milliunit fields, formatted-twins of fields already in `goal_summary`/`goal_status`, or goal attributes whose values are fully captured in the derived strings (e.g. `goal_under_funded`, `goal_overall_left`, `goal_target`, `goal_overall_funded`, `goal_snoozed_at`, `goal_day`, `goal_cadence`, `goal_cadence_frequency`, `goal_creation_month`, `goal_months_to_budget`, `goal_needs_whole_amount`). All 16 dropped fields remain reachable via the `data://categories/{id}/full` drill-in (`full_details`).

#### Scenario: No goal yields empty goal model
- **WHEN** `MCPCategoryGoal.from_ynab` is called with a `ynab.Category` whose `goal_type` is None
- **THEN** the returned instance has every field set to None, including `goal_summary` and `goal_status`

#### Scenario: TB goal explains target balance
- **WHEN** the raw category has `goal_type = "TB"` and a `goal_target_formatted` value
- **THEN** `goal_summary` contains the literal phrase "Target Category Balance" and references the formatted target amount
- **AND** `goal_status` reflects whether the category is currently under- or over-funded

#### Scenario: TBD goal explains target balance by date
- **WHEN** the raw category has `goal_type = "TBD"` and a `goal_target_date`
- **THEN** `goal_summary` contains "Target Category Balance by Date" and references the target date and formatted target amount

#### Scenario: MF goal explains cadence and refill vs set-aside
- **WHEN** the raw category has `goal_type = "MF"`
- **THEN** `goal_summary` describes the cadence in plain English (e.g. "every 2 months", "weekly", "every 2 years", "by <date>" when cadence is 0)
- **AND** the phrasing uses "Refill" when `goal_needs_whole_amount` is False and "Set aside" when True
- **AND** `_explain_goal_type` raises `ValueError` for any cadence value outside 0–14 or any missing required `goal_cadence_frequency`

#### Scenario: Weekly cadence uses frequency, not cadence*frequency
- **WHEN** the raw category has `goal_type = "MF"`, `goal_cadence = 2` (weekly), and `goal_cadence_frequency = 1`
- **THEN** `goal_summary` reads "every week" (or "weekly")
- **AND** it does NOT read "every 2 weeks" — cadence 2 means weekly, and `goal_cadence_frequency` is a multiplier (`2` would mean "every 2 weeks")

#### Scenario: Yearly cadence uses frequency, not cadence*frequency
- **WHEN** the raw category has `goal_type = "MF"`, `goal_cadence = 13` (yearly), and `goal_cadence_frequency = 1`
- **THEN** `goal_summary` reads "every year" (or "yearly")
- **AND** it does NOT read "every 13 years" — cadence 13 means yearly, and `goal_cadence_frequency` is a multiplier (`2` would mean "every 2 years")

#### Scenario: Biweekly cadence is "every 2 weeks"
- **WHEN** the raw category has `goal_type = "MF"`, `goal_cadence = 2` (weekly), and `goal_cadence_frequency = 2`
- **THEN** `goal_summary` reads "every 2 weeks"
- **AND** the math is `goal_cadence_frequency` (2) weeks apart, NOT `goal_cadence * goal_cadence_frequency` (4) weeks apart

#### Scenario: Every-N-months cadence is cadence-1
- **WHEN** the raw category has `goal_type = "MF"`, `goal_cadence = 3`
- **THEN** `goal_summary` reads "every 2 months" (cadence 3 means "every (3-1) months")
- **AND** `goal_cadence_frequency` is ignored for cadence values 3-12 and 14

#### Scenario: Every 2 years cadence ignores frequency
- **WHEN** the raw category has `goal_type = "MF"`, `goal_cadence = 14`
- **THEN** `goal_summary` reads "every 2 years"
- **AND** `goal_cadence_frequency` is ignored

#### Scenario: NEED goal explains plan-your-spending
- **WHEN** the raw category has `goal_type = "NEED"`
- **THEN** `goal_summary` contains "Plan Your Spending"

#### Scenario: Unknown goal type is labelled but not an error
- **WHEN** the raw category has an unrecognised `goal_type`
- **THEN** `goal_summary` returns `"Unknown goal type: {raw.goal_type}"` and `from_ynab` does not raise

#### Scenario: Lean MCPCategoryGoal exposes only 5 fields
- **WHEN** `MCPCategoryGoal.model_fields` is inspected
- **THEN** exactly 5 fields are present: `goal_type`, `goal_target_date`, `goal_percentage_complete`, `goal_summary`, `goal_status`
- **AND** none of the following fields exist on the lean model: `goal_needs_whole_amount`, `goal_day`, `goal_cadence`, `goal_cadence_frequency`, `goal_creation_month`, `goal_target`, `goal_under_funded`, `goal_overall_funded`, `goal_overall_left`, `goal_snoozed_at`, `goal_target_formatted`, `goal_under_funded_formatted`, `goal_overall_funded_formatted`, `goal_overall_left_formatted`, `goal_months_to_budget`

#### Scenario: Dropped fields remain reachable via full_details
- **WHEN** the LLM reads `data://categories/{id}/full`
- **THEN** `full_details` contains all the fields the lean `MCPCategoryGoal` dropped: `goal_under_funded`, `goal_overall_funded`, `goal_overall_left`, `goal_snoozed_at`, `goal_target`, `goal_creation_month`, `goal_cadence`, `goal_cadence_frequency`, `goal_needs_whole_amount`, `goal_day`, and the four `*_formatted` companions
- **AND** those fields are reachable in the drill-in dict by their original YNAB field names
