# Tool proposal

## Positioning

Most existing YNAB MCPs — [rgarcia/ynab-mcp-server](https://github.com/rgarcia/ynab-mcp-server), [Jtewen/ynab-mcp](https://github.com/Jtewen/ynab-mcp), [calebl/ynab-mcp-server](https://github.com/calebl/ynab-mcp-server), [mattweg/ynab-mcp](https://github.com/mattweg/ynab-mcp) — are CRUD wrappers around the YNAB OpenAPI. They surface raw data but do not compute budget health.

This MCP is an **opinionated budget-health compute layer** on top of YNAB HTTP streaming. Two differentiators:

- **HTTP streaming responses** — lower latency for multi-tool aggregates (no other YNAB MCP streams)
- **Compute, not CRUD** — every tool answers a "what does this mean / what should I do" question, not "fetch me row X"

## Proposed tools

Eight tools, each derived from a documented white-space gap. Every one is compute, not CRUD.

### 1. `goal_feasibility`
- **User question**: "Can I achieve my vacation savings goal by August?"
- **Input**: a category with a goal target + target date
- **Output**: verdict (on track / at risk / off track), required monthly contribution, delta vs current pace
- **Maps to**: white-space #1, #6

### 2. `simulate_expense`
- **User question**: "What if I add a $200/mo car payment?"
- **Input**: expense amount + frequency (one-time or recurring) + start date
- **Output**: per-category impact, which goals slip, expected change in Ready to Assign. Suggestion to reassign budget from another category based on previous expense.
- **Maps to**: white-space #2, #3, #4

### 3. `pacing_report`
- **User question**: "Am I on track this month?"
- **Input**: current month (default) + optional historical window
- **Output**: per-category pace vs target, on-track / at-risk / over verdict
- **Maps to**: white-space #5, #9

### 4. `age_of_money_trend`
- **User question**: "Is my budgeting hygiene improving over time?"
- **Input**: window (3 / 6 / 12 months)
- **Output**: AoM series + slope
- **Maps to**: white-space #14

### 5. `idle_cash_diagnosis`
- **User question**: "Where is my money sitting idle?"
- **Input**: current month snapshot
- **Output**: Ready to Assign surplus, total overfunded, suggested sweep targets
- **Maps to**: white-space #12, #13

### 6. `reallocation_plan`
- **User question**: "How should I distribute extra money across categories this month?"
- **Input**: available amount + priority categories
- **Output**: per-category suggestion with rationale (goal impact, urgency)
- **Maps to**: white-space #7

### 7. `subscription_goal_impact`
- **User question**: "If I cut my streaming subscriptions, which goals would unblock?"
- **Input**: expense amount and category
- **Output**: per-subscription goal-unblocking impact
- **Maps to**: white-space #11

### 8. `anomaly_narration`
- **User question**: "Why is this category over budget?"
- **Input**: category + current month
- **Output**: outlier explanation (one-off vs subscription creep vs lifestyle shift)
- **Maps to**: white-space #9

## MCP prompts

In addition to the 8 tools, expose a small set of **MCP prompts** — pre-written message templates that guide client-side LLMs through specific budget-health workflows. Prompts ground the model in the right tool(s), reduce hallucination, and are discoverable in MCP clients (Claude Desktop, etc.). This is the same primitive `jamiew/monarch-mcp` uses for its `analyze_spending`, `budget_review`, and `financial_health_check` prompts.

Each prompt names a workflow, declares which tool(s) it should call, and frames the response format. A prompt can compose multiple tools.

### Per-tool prompts

| Prompt | Tool(s) | Workflow |
|---|---|---|
| `review_goal_progress` | `goal_feasibility` | Walk every category with a goal; emit a verdict per goal; focus the narrative on at-risk and off-track |
| `simulate_what_if` | `simulate_expense` | Take a user-proposed expense (one-time or recurring), run the simulation, present per-category impact and which goals slip |
| `monthly_health_check` | `pacing_report` | Per-category pace vs target for the current month; surface at-risk and over-budget categories first |
| `hygiene_review` | `age_of_money_trend` | 3 / 6 / 12-month AoM trend with slope and qualitative interpretation |
| `idle_cash_review` | `idle_cash_diagnosis` | Surplus in Ready to Assign, total underfunded, suggested sweep targets |
| `reallocate_funds` | `reallocation_plan` | Given an available amount and any priority categories, distribute and explain trade-offs |
| `subscription_audit` | `subscription_goal_impact` | For each recurring subscription, what unblocks if it's cut |
| `investigate_overrun` | `anomaly_narration` | Explain why a category is over budget; classify as one-off, subscription creep, or lifestyle shift |

### Cross-cutting prompts

These compose multiple tools for higher-value workflows:

- **`budget_health_overview`** — chains `pacing_report` + `age_of_money_trend` + `idle_cash_diagnosis`; the model's opening screen for a session. Answers "how am I doing right now?" in one pass.
- **`goal_planning_session`** — chains `goal_feasibility` → `reallocation_plan` → `simulate_expense`; the user names a goal they're trying to hit, the prompt helps them choose between funding more or trimming elsewhere. The closest thing to a "YNAB coach" workflow and the most defensible against the competitor CRUD wrappers.

### How prompts interact with the user study

Prompts directly address the **researcher-as-routing-engine** limitation flagged in `docs/research.md` — real usage depends on the LLM's tool-selection quality, which free-form Q&A does not exercise. A second-leg study could A/B test free-form vs prompt-invoked workflows on the same seeded budgets, measuring whether prompt-guided sessions produce higher task-success rates.

Prompts also give the Wizard-of-Oz study a more realistic routing layer: the researcher invokes a named prompt instead of picking a tool manually, which mirrors how a real MCP client would use the system.

## Visualisation

- Graphs of spending
- Graphs of projection

Tools that emit a time series or comparison series should expose a structured payload (label + series + unit) so a client can render without re-parsing. The MCP ships the data; the visual layer is the client's responsibility.

## Out of scope (confirming boundary)

- Bulk data hygiene (categorization cleanup, duplicate detection) — better as a separate AI-driven cleanup automation
- General-purpose transaction CRUD
- Cross-account aggregation outside YNAB — no Plaid integration
