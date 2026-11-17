# F05 — Budgets & Goals

| Item | Value |
|------|-------|
| PRD | §6.5 (BG-1 … BG-5) |
| Priority | P0: BG-1, BG-2. P1: BG-3, BG-4. P2: BG-5 |
| Milestone | M4 |
| Pages | `budgets`, budget widget on `home` |
| Tech refs | [03 § 6](../tech/03-dataflow.md#6-outbox--notifications--budget-alerts), [04 § 3.4](../tech/04-database-schema.md#34-recurring-budgets-goals), [05 § 2.5](../tech/05-api-design.md#25-budgets--goals-f05) |

## 1. Summary

Each ledger can have a monthly total budget and per-category budgets. Progress shows on the home screen and the budgets page. When spending crosses 80 % or 100 % (configurable), ledger members get an in-app notification and an optional email. Rollover carries unused amounts forward. Savings goals track progress toward a target amount linked to an account.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-05.1 | organizer | a NT$25,000 monthly household budget | we know our limit |
| US-05.2 | organizer | a separate dining budget | I can control the category we overspend on |
| US-05.3 | member | a warning when we reach 80 % | we can slow down before it is too late |
| US-05.4 | saver | unused budget to roll over | saving this month helps next month |
| US-05.5 | young professional | a "Japan trip NT$60,000 by June" goal | I stay motivated |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Priority | Acceptance criteria |
|----|-------------|-----|----------|---------------------|
| F05-1 | Ledger & category budgets | BG-1 | P0 | One total budget (`category_id NULL`) plus optional budgets per **top-level** category (children's spending is included). Changing an amount creates a new `effective_from` row from the current period, so history stays accurate. The period is the calendar month, starting on the user's `month_start_day` for personal ledgers or the group owner's for shared ledgers. |
| F05-2 | Progress display | BG-2 | P0 | Home shows the total budget bar (spent / limit, % and remaining days). The budgets page lists each budget with a bar colored green (< 80 %), amber (80–99 %), and red (≥ 100 %), plus "safe to spend per day" = remaining ÷ remaining days. |
| F05-3 | Threshold alerts | BG-2 | P0 | Default thresholds are `{80, 100}`, editable per budget (1–3 values, 1–150 %). Each threshold alerts **at most once per period** (`budget_alert_log`). Recipients are all ledger members (personal ledger: the owner). The alert appears in-app within 1 minute of the crossing transaction, plus an email if enabled. Deleting the transaction afterwards does not resend. |
| F05-4 | Rollover | BG-3 | P1 | If enabled, `limit(period) = amount + max(0, limit(prev) − spent(prev))`, carried for at most 1 period (no unbounded accumulation). Overspend does **not** reduce the next period (v1). |
| F05-5 | Savings goals | BG-4 | P1 | Goal = name, target, currency, optional deadline, optional linked account. Saved amount = linked account balance (or manual contributions recorded as transfers to the account). Projected completion = linear projection of the last 90 days' net inflow. Achieving it sets `achieved_at` and sends a celebration notification. |
| F05-6 | Custom periods | BG-5 | P2 | Weekly and payday-to-payday periods (`period = 'weekly'/'custom'`). |

## 4. Business rules

1. Only `expense` transactions count toward budgets. Transfers, settlements, adjustments, and income are excluded.
2. In shared ledgers, the **full** expense amount counts toward the ledger budget (not per-member shares).
3. A budget's spend uses `ledger_amount_minor` and `occurred_on` within the period.
4. Viewers can see budgets but cannot change them.
5. A category budget may exceed the total budget. The UI warns but allows it.

## 5. UI (Streamlit)

```
budgets — November (Household)
┌──────────────────────────────────────┐
│ Total     18,420 / 25,000   74 %     │
│ ███████████████░░░░░ safe/day 470    │
│ 🍽 Dining   4,900 / 5,000   98 % ⚠   │
│ 🛒 Groceries 6,100 / 9,000  68 %     │
│ 🚇 Transport 1,200 / 2,000  60 %     │
│ [+ Add category budget]  [Edit]      │
└──────────────────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `budgets.ledger`, `budgets.period` | selectors |
| `budgets.add`, `budgets.edit.<id>`, `budgets.delete.<id>` | dialog actions |
| `budgets.form.amount`, `budgets.form.category`, `budgets.form.rollover`, `budgets.form.thresholds` | form |
| `home.budget_bar` | container on home |
| `goals.add`, `goals.row.<id>` | goals section (P1) |

Bars use `st.progress` with text. Values over 100 % are clamped to 1.0 with a red "over by NT$x" caption.

## 6. Data & API

- Tables: `budgets`, `budget_alert_log`, `goals`, `notifications`, `outbox_events`, `transactions`, `categories`.
- Commands: `SetBudget`, `DeleteBudget`, `EvaluateBudgetAlerts` (worker), `CreateGoal`, `UpdateGoal`, `DeleteGoal`.
- Queries: `GetBudgetProgress`, `GetGoalProgress`.
- Domain: `BudgetEvaluator.crossed(prev_spent, new_spent, limit, thresholds) -> list[int]`.

## 7. Performance

`GetBudgetProgress` is **one** statement for all budgets of a ledger:

```sql
WITH active AS (
    SELECT DISTINCT ON (category_id) *
    FROM budgets
    WHERE ledger_id = :ledger_id AND effective_from <= :period_start
      AND (effective_to IS NULL OR effective_to > :period_start)
    ORDER BY category_id, effective_from DESC
),
spend AS (
    SELECT COALESCE(c.parent_id, c.id) AS top_id, SUM(t.ledger_amount_minor) AS spent
    FROM transactions t JOIN categories c ON c.id = t.category_id
    WHERE t.ledger_id = :ledger_id AND t.deleted_at IS NULL AND t.type = 'expense'
      AND t.occurred_on >= :period_start AND t.occurred_on < :period_end
    GROUP BY ROLLUP (COALESCE(c.parent_id, c.id))       -- NULL row = ledger total
)
SELECT a.id, a.category_id, a.amount_minor, a.rollover, a.alert_thresholds,
       COALESCE(s.spent, 0) AS spent_minor
FROM active a
LEFT JOIN spend s ON s.top_id IS NOT DISTINCT FROM a.category_id;
```

Rollover needs the previous period's spend. The same statement is run with the date range extended to the previous period and a `FILTER` clause per period. It stays a single statement. The worker evaluates alerts for **all** ledgers touched by an outbox batch with this query using `ledger_id = ANY(:ids)`.

## 8. Test checklist

- [ ] Unit: `BudgetEvaluator` crossing logic (exactly at threshold, jump over two thresholds at once, decrease below).
- [ ] Unit: rollover cap at one period; overspend not carried.
- [ ] Integration: alert sent once per threshold per period even when the worker reprocesses events.
- [ ] Integration: query budget = 1 with 1 and 20 budgets.
- [ ] Playwright: add expenses until 80 % → notification appears in the bell.
