# F06 — Reports & Insights

| Item | Value |
|------|-------|
| PRD | §6.6 (RP-1 … RP-7) |
| Priority | P0: RP-1, RP-2, RP-3. P1: RP-4, RP-5, RP-6. P2: RP-7 |
| Milestone | M4 |
| Pages | `reports` (tabs), summary cards on `home` |
| Tech refs | [05 § 2.6](../tech/05-api-design.md#26-reports-f06), [06 § 5.4–5.5](../tech/06-query-performance.md#54-category-breakdown-with-parent-roll-up-1-statement) |

## 1. Summary

Clear answers to "where did our money go?": a monthly overview compared with last month, a category breakdown with drill-down, multi-month trends, per-member spending in shared ledgers, a daily calendar heatmap, an automatic monthly summary, and a year-in-review. Every chart is driven by **one aggregate SQL statement** and cached by ledger revision.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-06.1 | user | this month's income, spending, and net vs. last month | I know if we're doing better |
| US-06.2 | user | spending by category, tapping into a category to see its transactions | I find the leaks |
| US-06.3 | user | the 12-month trend of dining spend | I see habits change |
| US-06.4 | couple | how much each of us paid for the household | contributions feel fair |
| US-06.5 | user | a monthly summary without opening the app | I stay aware with no effort |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Priority | Acceptance criteria |
|----|-------------|-----|----------|---------------------|
| F06-1 | Monthly overview | RP-1 | P0 | Three `st.metric` tiles (Income, Expense, Net) with delta vs. the previous period (% and absolute). Period selector: month (default), previous/next arrows, custom range. Uses the user's `month_start_day`. |
| F06-2 | Category breakdown | RP-2 | P0 | Plotly donut (top 8 + "Others") plus a ranked list with amount, %, and transaction count. Selecting a top-level category expands its subcategories. "View transactions" opens the `transactions` page pre-filtered (category + period). Toggle expense/income. |
| F06-3 | Trend | RP-3 | P0 | Bar chart of monthly totals over 6 or 12 months. Optional category multiselect for line overlays. Months with no data show 0. |
| F06-4 | Per-member spending | RP-4 | P1 | Shared ledgers only: for each member, **paid** total and **share** total in the period, as grouped bars. |
| F06-5 | Calendar | RP-5 | P1 | Month heatmap of daily spend (Plotly heatmap 7 × weeks). Selecting a day lists that day's transactions. |
| F06-6 | Monthly summary | RP-6 | P1 | On day 1 at 09:00 local time, the worker creates a `monthly_summary` notification (and email if enabled): totals, top 3 categories, the biggest change vs. the previous month ("Dining −12 %"), and budget outcome. Dedupe key `monthly_summary:<yyyy-mm>`. |
| F06-7 | Year in review | RP-7 | P2 | Annual totals, top categories and merchants, busiest month, longest logging streak. |

## 4. Business rules

1. Reports include only `expense` and `income`. Transfers, settlements, and adjustments never count as spending or income.
2. Amounts use `ledger_amount_minor` (ledger currency).
3. Report scope is one ledger at a time. An "All my spending" view (personal + my shares of shared ledgers) is a P1 option: it combines personal-ledger expenses with the actor's `share_minor` in shared ledgers, in one `UNION ALL` statement.
4. Soft-deleted transactions are excluded.
5. Percentages are rounded to one decimal. Displayed shares are rounded so they sum to 100 %, using largest remainder.

## 5. UI (Streamlit)

```
reports — Household · Nov 2026  ◀ ▶
[Overview] [Categories] [Trend] [Members] [Calendar]
┌────────┬────────┬────────┐
│Income  │Expense │Net     │
│ 92,000 │ 48,310 │ 43,690 │
│ +0 %   │ −8.2 % │ +9.6 % │
└────────┴────────┴────────┘
   ◔ donut            1. 🛒 Groceries 12,400 25.7 %
                      2. 🏠 Housing   11,000 22.8 %
                      3. 🍽 Dining     6,900 14.3 %
```

| Widget key | Element |
|------------|---------|
| `reports.ledger`, `reports.period.prev`, `reports.period.next`, `reports.period.custom` | scope |
| `reports.tabs` | `st.tabs` |
| `reports.kind` | `st.segmented_control` expense / income |
| `reports.category.<id>` | ranked-list row → expand / drill-down |
| `reports.trend.months`, `reports.trend.categories` | trend controls |

Charts use one shared Plotly theme (`components/charts.py`) for consistency in light and dark mode. Category colors come from `categories.color`.

## 6. Data & API

- Tables: `transactions`, `categories`, `transaction_splits` (members), `notifications`.
- Queries: `GetMonthlyOverview`, `GetCategoryBreakdown`, `GetSpendingTrend`, `GetMemberSpending`, `GetDailyCalendar`.
- Worker: `SendMonthlySummaries` (batched by 200 users; each batch uses set-based queries with `ledger_id = ANY(:ids)`).

## 7. Performance

| Query | Statements | Technique |
|-------|-----------|-----------|
| Overview | 1 | `SUM(...) FILTER (WHERE type = … AND occurred_on in period)` for current and previous period in one scan |
| Category breakdown | 1 | `GROUPING SETS` for parent and child totals ([06 § 5.4](../tech/06-query-performance.md#54-category-breakdown-with-parent-roll-up-1-statement)) |
| Trend | 1 | `generate_series` of months `LEFT JOIN` grouped sums by `date_trunc('month', occurred_on)` |
| Members | 1 | paid CTE + share CTE (same shape as balances) restricted to the period |
| Calendar | 1 | `GROUP BY occurred_on` |

All of these are wrapped in `st.cache_data` keyed by `(query, ledger_id, revision, params)`. Switching tabs back and forth costs only the revision lookup. Target p95 ≤ 150 ms uncached at 100 k transactions per ledger.

## 8. Test checklist

- [ ] Integration: each report query returns correct totals on a fixed fixture ledger (golden numbers).
- [ ] Integration: transfers, settlements, and adjustments are excluded.
- [ ] Integration: query budget = 1 per report with 1 and 5,000 transactions.
- [ ] Unit: percentage rounding sums to 100.0.
- [ ] AppTest: drill-down sets the transaction filter in session state and switches page.
