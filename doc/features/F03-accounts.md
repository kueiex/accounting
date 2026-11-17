# F03 — Accounts

| Item | Value |
|------|-------|
| PRD | §6.3 (AC-1 … AC-5) |
| Priority | P0: AC-1, AC-2. P1: AC-3, AC-4, AC-5 |
| Milestone | M2 (P0), M6 (P1) |
| Pages | `accounts` (list + detail dialog) |
| Tech refs | [04 § 3.2](../tech/04-database-schema.md#32-ledgers-accounts-categories-tags), [05 § 2.3](../tech/05-api-design.md#23-accounts-f03), [06 § 5.3](../tech/06-query-performance.md#53-account-balances-for-all-of-a-users-accounts-1-statement) |

## 1. Summary

Accounts represent where money lives: cash, bank, credit card, e-wallet (LINE Pay, JKOPAY, EasyCard, iPASS), investment (balance only), and loan. Each account belongs to one user and shows a running balance derived from transactions. Credit cards add statement and due-day reminders. Users can see their net worth and optionally share specific accounts (e.g. a joint bank account) with the family.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-03.1 | user | to list my cash, bank, card, and e-wallet accounts with balances | I know what I have |
| US-03.2 | user | to correct a balance to match my bank app | small gaps don't break my books |
| US-03.3 | credit-card user | a reminder before my card payment is due | I avoid late fees |
| US-03.4 | user | to see my net worth | I see long-term progress |
| US-03.5 | couple with a joint account | both of us to see and use the joint account | shared spending uses the right account |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Priority | Acceptance criteria |
|----|-------------|-----|----------|---------------------|
| F03-1 | Account CRUD & archive | AC-1 | P0 | Create with name, type, currency, opening balance (liabilities entered as amount owed, stored negative). Edit uses `expected_version`. Archive hides the account from pickers but keeps history. An archived account with a non-zero balance shows a warning first. Drag-free reorder via ▲▼ buttons (`ReorderAccounts`). |
| F03-2 | Running balance | AC-2 | P0 | Balance = opening + Σ inflows − Σ outflows over non-deleted transactions ([04 § 4](../tech/04-database-schema.md#4-derived-values-never-stored)). The list shows all balances from **one** query. Account detail shows the last 50 transactions for the account (keyset pagination). |
| F03-3 | Balance adjustment | AC-2 | P0 | "Adjust balance" asks for the actual balance and creates an `adjustment` transaction for the difference (sign recorded). A zero difference creates nothing. Adjustments are excluded from income and expense reports. |
| F03-4 | Credit-card cycle & reminder | AC-3 | P1 | Statement day and due day (1–31, clamped to month end). The card shows "Current cycle spend" (statement-day window). Notification `card_due` 3 days before the due day (worker, dedupe key `card_due:<account>:<yyyy-mm>`). |
| F03-5 | Net worth | AC-4 | P1 | Assets − liabilities over own accounts with `include_in_net_worth`, converted to the default currency at the latest FX rate. A 12-month net-worth trend comes from month-end balances (one window-function query). |
| F03-6 | Share account with family | AC-5 | P1 | The owner shares an account with their group as `read` or `write`. Members with `write` may pick it as the paying account in shared ledgers. The account balance becomes visible to group members. Unsharing does not change history. |

## 4. Business rules

1. Account currency is immutable once transactions reference the account.
2. A transaction in a shared ledger debits the **payer's** account. Members never see another member's private account name. Shared-ledger lists show "Paid by Jason" instead of the account name unless the account is shared.
3. Investment accounts track a manually adjusted balance only (no holdings or prices; PRD non-goal).
4. Loan accounts are liabilities. Payments are transfers from a bank or cash account to the loan account.
5. Deleting an account is not offered; only archive. (Hard delete happens only through account deletion, F07.)

## 5. UI (Streamlit)

```
accounts
┌──────────────────────────────────┐
│ Net worth  NT$ 312,400  (P1)     │
├──────────────────────────────────┤
│ 💵 Cash                  2,150   │
│ 🏦 Cathay Bank          85,200   │
│ 💳 CTBC Card           -12,480   │
│    due in 6 days                 │
│ 📱 LINE Pay              1,030   │
│ [+ Add account]  [Show archived] │
└──────────────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `accounts.add` | button → `st.dialog` with form (`accounts.form.*`) |
| `accounts.row.<id>` | row button → detail dialog |
| `accounts.detail.adjust`, `accounts.detail.archive`, `accounts.detail.share` | dialog actions |
| `accounts.move_up.<id>`, `accounts.move_down.<id>` | reorder |
| `accounts.show_archived` | `st.toggle` |

## 6. Data & API

- Tables: `accounts`, `account_shares`, `transactions` (derived balances), `fx_rates`, `notifications`.
- Commands: `CreateAccount`, `UpdateAccount`, `ArchiveAccount`, `UnarchiveAccount`, `ReorderAccounts`, `AdjustAccountBalance`, `ShareAccount`, `UnshareAccount`.
- Queries: `ListAccountsWithBalances`, `GetNetWorth`, plus `ListTransactions` filtered by `account_ids`.

## 7. Performance

- `ListAccountsWithBalances`: 1 statement using LATERAL sums over covering indexes. Index-only scans keep it ≤ 20 ms for 20 accounts × 50 k transactions.
- Reorder: one `UPDATE accounts SET sort_order = v.ord FROM (VALUES …) v WHERE accounts.id = v.id`.
- Net-worth trend: one statement with `generate_series` month-ends and a windowed running sum, not one query per month.

## 8. Test checklist

- [ ] Unit: balance formula for every transaction type, including cross-currency transfers.
- [ ] Unit: adjustment creates the right sign; zero diff creates nothing.
- [ ] Integration: balance query budget = 1 with 1 and 20 accounts.
- [ ] Authz: a member cannot read another member's unshared account (returns `NotFound`).
- [ ] AppTest: archive warning appears when the balance is non-zero.
