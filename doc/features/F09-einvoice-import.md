# F09 — Taiwan E-Invoice Import

| Item | Value |
|------|-------|
| PRD | §6.2 TX-8, PRD flow 7.3 |
| Priority | P1 (first item of v1.x) |
| Milestone | M6 |
| Pages | `settings` › E-invoice (link carrier), `inbox` (review drafts) |
| Tech refs | [03 § 8](../tech/03-dataflow.md#8-e-invoice-import-p1), [04 § 3.5](../tech/04-database-schema.md#35-e-invoice-fx-notifications-outbox-preferences), [05 § 2.9, § 3](../tech/05-api-design.md#29-e-invoice-f02-tx-8-p1) |

## 1. Summary

Many purchases in Taiwan already produce a cloud e-invoice stored on the shopper's **mobile barcode carrier** (手機條碼載具). After the user links their carrier, Kinbook pulls new invoices from the Ministry of Finance (MOF) e-invoice platform every few hours. It turns them into **draft transactions** with merchant, amount, date, line items, and a suggested category. The user reviews them in an Inbox and accepts them in bulk. This is the main automation lever against "I stopped logging".

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-09.1 | reluctant partner (Jason) | my convenience-store and supermarket purchases to appear automatically | I barely need to type |
| US-09.2 | user | to confirm drafts in one swipe-like action | reviewing is faster than typing |
| US-09.3 | user | drafts to land in the right ledger (e.g. supermarket → Household) | shared spending is split without extra steps |
| US-09.4 | user | an invoice I already logged by hand not to be counted twice | totals stay right |

## 3. Requirement breakdown

| ID | Sub-feature | Acceptance criteria |
|----|-------------|---------------------|
| F09-1 | Link carrier | User enters the carrier code (format `/` + 7 chars of `0-9 A-Z . - +`) and verification code. `LinkCarrier` verifies against MOF before saving. The verification code is stored encrypted (Fernet). The user picks a default ledger and account for drafts. Unlink deletes credentials and pending drafts (accepted transactions stay). |
| F09-2 | Scheduled sync | Worker syncs each active carrier every 6 h (± 30 min jitter). The first sync backfills the current and previous month. MOF errors set backoff. Invalid credentials set `status = 'invalid'` and notify the user. |
| F09-3 | Draft creation | Each new invoice (unique by `carrier_id, invoice_number`) becomes an `einvoices` row with `status = 'pending'`, line items (JSON), and a suggested category. One `einvoice_new` notification per sync with the count (not one per invoice). |
| F09-4 | Category suggestion | Priority: (1) the user's own history for the same seller BAN or normalized seller name (most frequent category in 180 days); (2) a keyword rule table on seller name and items (e.g. 全聯 / 家樂福 → Groceries; 7-ELEVEN with only drink items → Drinks); (3) "Other". |
| F09-5 | Inbox review | The Inbox lists pending drafts newest first: date, seller, amount, top items, and suggested category (editable inline in `st.data_editor`), ledger, and account. Actions: **Accept selected**, **Accept all with suggestions**, **Ignore selected**. Accepting into a shared ledger applies the ledger's default split with the user as payer. |
| F09-6 | Duplicate guard | On accept, if a manual transaction exists in the target ledger with the same amount on the same date (± 1 day) created by the same user, the row shows "Possible duplicate" and needs explicit confirmation (or "Link to existing", which marks the invoice accepted with that `transaction_id`). |
| F09-7 | Badge | The Inbox nav item shows the pending count (one `COUNT` on `einvoices_inbox_idx`). |

## 4. Business rules

1. Kinbook never writes to the MOF platform. Access is read-only.
2. Invoices whose amount ≤ 0 (voided or returned) are not imported as expenses. A void of an already accepted invoice notifies the user to review the transaction.
3. Accepted transactions have `source = 'einvoice'`, `merchant = seller_name`, and line items summarized in `note` (≤ 500 chars).
4. Each user links their own carrier. Drafts are private to the user until they accept them into a shared ledger.
5. The feature requires an MOF AppID and API key held by Kinbook. Terms of use and rate limits must be confirmed with MOF before launch (open item).

## 5. UI (Streamlit)

```
inbox (6)
┌──────────────────────────────────────────────────────────┐
│ ☑ 11/16  全聯福利中心   1,284  [Groceries ▾] [Household ▾] │
│ ☑ 11/16  7-ELEVEN         65  [Drinks ▾]    [Personal ▾]  │
│ ☐ 11/15  家樂福         2,480  ⚠ possible duplicate        │
│ [Accept selected] [Accept all] [Ignore selected]          │
└──────────────────────────────────────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `einvoice.carrier_code`, `einvoice.verification`, `einvoice.default_ledger`, `einvoice.default_account`, `einvoice.link`, `einvoice.unlink` | settings tab |
| `inbox.table` | `st.data_editor` (select, category, ledger, account columns) |
| `inbox.accept_selected`, `inbox.accept_all`, `inbox.ignore_selected` | actions |
| `inbox.duplicate.<einvoice_id>.link` / `.confirm` | duplicate resolution |

## 6. Data & API

- Tables: `einvoice_carriers`, `einvoices`, `transactions`, `notifications`, `quick_add_preferences` (ledger preference per category also applies to drafts).
- Commands: `LinkCarrier`, `UnlinkCarrier`, `SyncCarrierInvoices` (worker), `AcceptInvoiceDrafts`, `IgnoreInvoiceDrafts`.
- Queries: `ListInvoiceInbox`.
- Port: `EInvoiceGateway` (adapter `MofEInvoiceGateway`).

## 7. Performance

- Sync job: up to 50 due carriers per tick, selected with `FOR UPDATE SKIP LOCKED` (multiple workers are safe). Per carrier there is 1 header call plus 1 detail call per new invoice (MOF API limit; HTTP, not DB). DB writes are one multi-row `INSERT … ON CONFLICT DO NOTHING` per carrier.
- Category suggestion for a batch uses **one** query: history aggregated by seller for all sellers in the batch (`seller_ban = ANY(:bans)`).
- `ListInvoiceInbox`: 1 statement (einvoices ⋈ categories on suggested slug for the default ledger), keyset paginated.
- `AcceptInvoiceDrafts` for n drafts: constant statements (validation lookups batched; transactions, splits, and history inserted multi-row; einvoices updated with `id = ANY`).
- Duplicate detection: one statement joining pending drafts with candidate transactions by `(ledger_id, ledger_amount_minor, occurred_on ± 1)`.

## 8. Test checklist

- [ ] Unit: carrier-code format validation.
- [ ] Unit: suggestion priority order (history > rules > Other).
- [ ] Integration (fake gateway): rerunning sync creates no duplicates.
- [ ] Integration: accepting 1 vs. 100 drafts uses the same number of statements.
- [ ] Integration: void invoice after acceptance triggers a review notification.
- [ ] Contract test against the MOF sandbox (manual, before release).
