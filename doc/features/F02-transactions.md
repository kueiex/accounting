# F02 — Recording Transactions

| Item | Value |
|------|-------|
| PRD | §6.2 (TX-1 … TX-7, TX-9 … TX-12). TX-8 is in [F09](F09-einvoice-import.md). |
| Priority | P0 core; P1 multi-currency, location, smart category, home-screen shortcut; P2 OCR, voice |
| Milestone | M2 |
| Pages | `add` (quick add), `transactions` (list, search, detail dialog, trash) |
| Tech refs | [03 § 2](../tech/03-dataflow.md#2-record-a-shared-expense-prd-flow-72-first-half), [04 § 3.3](../tech/04-database-schema.md#33-transactions), [05 § 2.2](../tech/05-api-design.md#22-transactions-f02), [06 § 5.1](../tech/06-query-performance.md#51-transaction-list-page-2-statements-for-any-page-size) |

## 1. Summary

The heart of the product: logging an expense in **≤ 5 seconds**. The quick-add page opens with the amount field focused. Category chips are ordered by recent use, and account and ledger come from smart defaults. Users can also record income and transfers, attach receipts, set up recurring entries, and search their history.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-02.1 | busy parent | to log "150, Lunch" in two taps after typing the amount | logging never feels like a chore |
| US-02.2 | user | the app to remember the account and ledger I usually use for a category | I rarely change defaults |
| US-02.3 | user | to record income and moves between my accounts | balances stay right |
| US-02.4 | user | to photograph a receipt and attach it | I can check it later |
| US-02.5 | user | rent and subscriptions logged automatically | I don't forget fixed costs |
| US-02.6 | user | to find "that Costco trip in March" quickly | I can answer money questions |
| US-02.7 | user | to undo an accidental delete | I don't lose data |
| US-02.8 | traveler (P1) | to log in JPY and see TWD in reports | trips are tracked correctly |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Status | Priority | Acceptance criteria |
|----|-------------|-----|--------|----------|---------------------|
| F02-1 | Quick-add form | TX-1 | Keep | P0 | **Given** the `add` page **when** it loads **then** the amount input is focused and category chips show the user's 8 most recent categories, followed by the ledger's remaining categories in `sort_order`. **When** the user enters an amount and taps a chip and Save **then** the transaction is saved with date = now (user TZ), account = last used, ledger = remembered preference for that category (else personal). Median time ≤ 5 s in usability tests. |
| F02-2 | Types: expense / income / transfer | TX-2 | Keep | P0 | `st.segmented_control` switches type. Transfer shows From / To account pickers and hides category. Saving a transfer with the same From and To fails with `transfer.same_account`. |
| F02-3 | Optional details | TX-3 | Keep | P0 | Expander "More": date/time, account, note (≤ 500), merchant, tags (multiselect + create), receipt (≤ 5 files, ≤ 10 MB each, JPEG/PNG/HEIC/PDF). |
| F02-4 | Location | TX-3 | Keep | P1 | Optional place text with browser geolocation (via the `streamlit-js-eval` component) only after explicit consent. |
| F02-5 | Edit / duplicate / delete / trash | TX-4 | Keep | P0 | Detail opens in `st.dialog`. Edit sends `expected_version`; a conflict shows the conflict dialog ([03 § 4](../tech/03-dataflow.md#4-edit-with-a-concurrency-conflict)). Delete is soft. Trash lists deleted items for 30 days with Restore. Duplicate copies all fields except date (set to now). |
| F02-6 | Recurring transactions | TX-5 | Keep | P0 | Frequencies: daily, weekly (weekday picker), monthly (day 1–31; 31 → last day), yearly, custom RRULE. Mode `auto` posts on the due date; `remind` creates a notification with Confirm / Skip. Editing a rule never changes already-posted transactions. |
| F02-7 | Multi-currency | TX-6 | Keep | P1 | Currency picker per transaction. `ledger_amount_minor` is computed from the `fx_rates` row for `occurred_on` (latest earlier rate if missing). The user may override the rate. Reports always use the ledger currency. |
| F02-8 | Home-screen shortcut | TX-7 | Adapted | P1 | `/add` is a standalone page that loads with minimal queries. Settings shows a how-to for "Add to Home Screen" on iOS and Android. |
| F02-9 | Receipt OCR | TX-9 | Keep | P2 | Photo → OCR adapter → prefilled draft (amount, date, merchant), shown for confirmation. |
| F02-10 | Voice entry | TX-10 | Keep | P2 | `st.audio_input` → speech-to-text adapter → rule-based parser (amount, category keyword, date words) → draft. |
| F02-11 | Smart category suggestion | TX-11 | Keep | P1 | When the merchant or note is typed, `SuggestCategory` returns the most frequent category for the same normalized merchant in the last 180 days (one aggregated query). The suggestion is preselected, never auto-saved. |
| F02-12 | Search & filter | TX-12 | Keep | P0 | Filters: text (trigram on note + merchant), date range, amount range, categories, accounts, tags, members (shared ledgers), types. Results are keyset-paginated, 50 per page, with "Load more". Filters persist in session state while navigating. |

## 4. Business rules

1. `amount_minor > 0`. The direction comes from `type`. Max amount is 99,999,999,999 minor units.
2. Expense and income require a category of the matching kind from the **same ledger**.
3. In a **personal** ledger: no splits; `payer_user_id = actor`.
4. In a **shared** ledger: payer defaults to the actor and may be any member with `can_write`. The split defaults to the ledger's `default_split_method` across all ledger members. Split rules are in [F04](F04-family-shared-ledgers.md).
5. An account used must belong to the payer (or be shared to the group with `write`, P1).
6. `occurred_on` is the date of `occurred_at` in the **actor's** time zone at the time of entry.
7. A transaction can be **moved** between the actor's personal ledger and a shared ledger (`MoveTransactionToLedger`). The category is re-mapped by slug, falling back to "Other". Moving into a shared ledger requires a split; moving out removes splits. Moving never happens implicitly.
8. Soft-deleted transactions are excluded from every balance, report, and budget. They are purged after 30 days.
9. Recurring rules are tied to a ledger. If the template's account or category is archived, the next occurrence is converted to `remind` mode and the user is notified.

## 5. UI (Streamlit)

```
add                                   transactions
┌──────────────────────────────┐      ┌──────────────────────────────┐
│ [Expense|Income|Transfer]    │      │ 🔍 [search……]   [Filters ▾]  │
│ NT$ [        150 ]           │      │ Nov 17                       │
│ 🍱Lunch 🚇MRT 🛒Groceries …   │      │  🍱 Lunch        -150  Cash  │
│ Ledger: (Personal|Household) │      │  🚇 MRT           -35        │
│ ▸ More (date, account, note, │      │ Nov 16                       │
│   tags, receipt, repeat)     │      │  🛒 Groceries  -2,480  👥2 📎 │
│ [        Save        ]       │      │ [Load more]   🗑 Trash        │
└──────────────────────────────┘      └──────────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `add.type` | `st.segmented_control` |
| `add.amount` | `money_input` (`st.number_input`, step 1, TWD without decimals) |
| `add.category` | `st.pills` (single select) |
| `add.ledger` | `st.segmented_control` (shown only when the user has a shared ledger) |
| `add.payer`, `add.split.method`, `add.split.<user_id>` | shown when the ledger is shared |
| `add.more` | `st.expander` |
| `add.date`, `add.time`, `add.account`, `add.to_account`, `add.note`, `add.merchant`, `add.tags`, `add.receipt`, `add.repeat` | detail widgets |
| `add.save` | primary `st.button` |
| `tx.search`, `tx.filters.*`, `tx.row.<id>`, `tx.load_more`, `tx.trash` | list page |
| `tx.detail.edit`, `tx.detail.delete`, `tx.detail.duplicate`, `tx.detail.move` | detail dialog |

- The quick-add form is an `@st.fragment`, so typing and chip selection rerun only the fragment.
- After Save: `st.toast`, then reset amount, note, and tags. Type, ledger, and account keep their values (fast repeated entry).

## 6. Data & API

- Tables: `transactions`, `transaction_splits`, `transaction_tags`, `tags`, `attachments`, `transaction_history`, `recurring_rules`, `recurring_occurrences`, `quick_add_preferences`, `fx_rates`.
- Commands: `RecordTransaction`, `UpdateTransaction`, `DeleteTransaction`, `RestoreTransaction`, `DuplicateTransaction`, `MoveTransactionToLedger`, `AttachReceipt`, `RemoveAttachment`, `CreateRecurringRule`, `UpdateRecurringRule`, `PauseRecurringRule`, `DeleteRecurringRule`, `ConfirmRecurringOccurrence`, `RunDueRecurringRules`.
- Queries: `GetQuickAddContext`, `SuggestCategory`, `ListTransactions`, `GetTransactionDetail`, `ListTrash`.

## 7. Performance

| Operation | Budget | How |
|-----------|--------|-----|
| Open `add` page | 4 statements (`GetQuickAddContext`) | Accounts, ledgers, recent categories (index `transactions_creator_recent_idx`), preferences |
| Save | ≤ 9 statements | Multi-row inserts for splits and tags |
| List page | 2 statements for any page size | [06 § 5.1](../tech/06-query-performance.md#51-transaction-list-page-2-statements-for-any-page-size) |
| Detail dialog | 3 statements | tx ⋈ names; splits + tags + attachments (batch); history ⋈ users |
| Search p95 | ≤ 150 ms at 100 k rows | `transactions_search_trgm_idx` + ledger predicate |

## 8. Test checklist

- [ ] Hypothesis: any amount and member set → split shares sum exactly to the amount; remainder goes to the payer first.
- [ ] Unit: shape rules (transfer same account, category kind mismatch, personal ledger with splits).
- [ ] Unit: recurrence edge cases (31st in February, leap year, DST-free Asia/Taipei, `until` boundary).
- [ ] Integration: `RunDueRecurringRules` is idempotent when run twice for the same day.
- [ ] Integration: query budgets with 1 and 200 rows.
- [ ] AppTest: Save with only amount + chip works; disabled Save without an amount.
- [ ] Playwright: median of 20 scripted quick adds ≤ 5 s including page load.
