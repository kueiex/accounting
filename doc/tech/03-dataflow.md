# 03 — Data Flow

## 1. Layered flow (every interaction)

```mermaid
flowchart LR
    W["Streamlit widget<br/>(key=…)"] -->|callback builds| C["Pydantic command"]
    C --> UC["Use case<br/>(application)"]
    UC -->|require_*| AZ["AccessPolicy"]
    UC -->|load / save| UOW["Unit of Work<br/>(1 DB transaction)"]
    UOW --> R["Repositories"] --> DB[("PostgreSQL")]
    UC --> DOM["Domain objects<br/>+ services"]
    UOW -->|same tx| OB["outbox_events"]
    UC -->|result DTO| W2["st.toast / rerun"]

    P["Page render"] -->|authz + revision| QS["Query service"] -->|1–2 SQL| DB
    QS -->|DTO| CACHE["st.cache_data<br/>key incl. revision"] --> P
```

- **Write path:** widget → command → use case → domain → UoW commit (data + revision bump + history + outbox, atomically).
- **Read path:** page → authz check → revision lookup → cached query service → DTO → widgets.
- **Async path:** worker polls the outbox → notifications, emails, budget alerts.

## 2. Record a shared expense (PRD flow 7.2, first half)

```mermaid
sequenceDiagram
    actor J as Jason (browser)
    participant UI as add.py (fragment)
    participant UC as RecordTransaction
    participant AZ as AccessPolicy
    participant D as Domain (Transaction, SplitAllocator)
    participant U as UnitOfWork / repos
    participant DB as PostgreSQL

    J->>UI: amount 2480, Groceries, ledger Household, paid by Jason, split equal
    UI->>UC: RecordTransactionCommand
    UC->>U: begin
    UC->>AZ: require_ledger_write(actor, ledger_id)
    AZ->>DB: SELECT ledger_members WHERE ledger_id, user_id
    UC->>U: ledgers.lock_and_bump(ledger_id)
    U->>DB: UPDATE ledgers SET revision = revision + 1 WHERE id = … RETURNING revision, currency
    UC->>U: validation lookups (account owner, category ledger, members)
    U->>DB: 1 SELECT (accounts ⋈ categories ⋈ ledger_members, filtered by ids)
    UC->>D: Transaction.record(..., SplitSpec.equal(members))
    D-->>UC: Transaction + Splits (1240, 1240) + TransactionRecorded event
    UC->>U: transactions.add(tx)
    U->>DB: INSERT transactions; INSERT transaction_splits (multi-row); INSERT transaction_tags
    U->>DB: INSERT transaction_history (action=create)
    U->>DB: INSERT outbox_events (TransactionRecorded)
    U->>DB: UPSERT quick_add_preferences
    UC->>U: commit
    UC-->>UI: RecordTransactionResult(id, revision)
    UI-->>J: toast "Saved", fragment resets
```

The ledger row lock (`UPDATE … revision + 1`) serializes concurrent writes **within one ledger**. This keeps revision numbers monotonic and costs nothing at family scale.

## 3. Read the dashboard (with cache)

```mermaid
sequenceDiagram
    actor M as Mei (browser)
    participant P as home.py
    participant AZ as AccessPolicy
    participant C as st.cache_data
    participant Q as DashboardQueries
    participant DB as PostgreSQL

    M->>P: open Home (rerun)
    P->>AZ: require_ledger_read(actor, ledger)
    Note over AZ: ledger membership is cached per session<br/>in session_state after first check
    P->>DB: SELECT revision FROM ledgers WHERE id = …
    P->>C: dashboard(actor, ledger, revision, month)
    alt cache hit (no writes since last render)
        C-->>P: DashboardDTO
    else cache miss
        C->>Q: get(actor, ledger, month)
        Q->>DB: month totals (1), budget progress (1), recent tx (1)
        Q-->>C: DashboardDTO
        C-->>P: DashboardDTO
    end
    P-->>M: render progress bar, list, charts
```

When Jason saves an expense, the revision becomes N+1. Mei's next rerun (her own interaction, or the 60-second polling fragment) misses the cache and sees fresh data.

## 4. Edit with a concurrency conflict

```mermaid
sequenceDiagram
    actor M as Mei
    actor J as Jason
    participant UC as UpdateTransaction
    participant DB as PostgreSQL
    M->>UC: update(tx, version=3, amount=2600)
    UC->>DB: UPDATE transactions SET …, version = 4 WHERE id = … AND version = 3
    DB-->>UC: 1 row
    UC-->>M: OK
    J->>UC: update(tx, version=3, note="Costco")
    UC->>DB: UPDATE … WHERE id = … AND version = 3
    DB-->>UC: 0 rows
    UC-->>J: ConcurrencyConflict(current_version=4, changed_by=Mei)
    Note over J: dialog: "Mei changed this a moment ago" → Reload / Re-apply my change
```

Every successful update writes a `transaction_history` row with a field-level diff, so members can see who changed what (FM-10).

## 5. Settle up (PRD flow 7.2, second half)

```mermaid
sequenceDiagram
    actor M as Mei
    participant P as family.py
    participant Q as FamilyQueries
    participant BC as BalanceCalculator
    participant UC as RecordSettlement
    participant DB as PostgreSQL
    P->>Q: balances(ledger)
    Q->>DB: 1 aggregated SQL (paid − owed per member)
    Q-->>P: [Mei −1240, Jason +1240]
    P->>BC: suggest_settlements(nets)
    BC-->>P: [Mei → Jason 1240]
    M->>P: tap "Settle up", choose account Cash
    P->>UC: RecordSettlementCommand(debtor=Mei, creditor=Jason, 1240, account=Cash)
    UC->>DB: INSERT transactions(type=settlement, payer=Mei, counterparty=Jason)<br/>INSERT transaction_splits(Jason, 1240), history, outbox, revision bump
    UC-->>P: OK → balances recompute to 0 / 0
```

## 6. Outbox → notifications & budget alerts

```mermaid
sequenceDiagram
    participant W as Worker (DispatchOutbox, every 10 s)
    participant DB as PostgreSQL
    participant BE as EvaluateBudgetAlerts
    participant N as NotificationService
    participant E as Resend
    W->>DB: SELECT … FROM outbox_events WHERE dispatched_at IS NULL ORDER BY id LIMIT 200 FOR UPDATE SKIP LOCKED
    W->>W: group events by type and ledger
    W->>BE: TransactionRecorded/Updated/Deleted for ledgers {L1, L2}
    BE->>DB: 1 SQL: budget progress for all budgets of {L1, L2} in current period
    BE->>DB: INSERT budget_alert_log … ON CONFLICT DO NOTHING RETURNING *
    BE->>N: thresholds newly crossed
    W->>N: shared-ledger events → recipients = ledger_members − actor (1 SQL for all ledgers)
    N->>DB: INSERT notifications (multi-row, dedupe_key)
    N->>E: send emails for users with email enabled (batched)
    W->>DB: UPDATE outbox_events SET dispatched_at = now() WHERE id = ANY(:ids)
```

On failure the worker increments `attempts` and `last_error` and leaves the event for retry with exponential backoff. After 10 attempts an alert fires.

## 7. Recurring transactions

```mermaid
flowchart LR
    T["Worker tick (15 min)"] --> L["Lock ≤ 500 due rules<br/>FOR UPDATE SKIP LOCKED"]
    L --> G["Domain: compute due occurrences<br/>(RRULE, until, next_run_on)"]
    G -->|mode = auto| M["Materialize transactions<br/>(splits from template)"]
    G -->|mode = remind| R["Notification:<br/>'Rent due — confirm?'"]
    M --> I["Bulk INSERT tx, splits<br/>INSERT recurring_occurrences ON CONFLICT DO NOTHING"]
    R --> I2["INSERT recurring_occurrences (tx NULL)<br/>INSERT notifications"]
    I --> N["UPDATE next_run_on; bump revisions; outbox"]
    I2 --> N
```

The `recurring_occurrences` primary key `(rule_id, occurrence_date)` makes the job safe to rerun after a crash. When a user confirms a "remind" notification, `ConfirmRecurringOccurrence` creates the transaction and fills in `recurring_occurrences.transaction_id`.

## 8. E-invoice import (P1)

```mermaid
sequenceDiagram
    actor U as User
    participant S as settings.py
    participant LC as LinkCarrier
    participant W as Worker (SyncCarrierInvoices)
    participant G as EInvoiceGateway (MOF API)
    participant DB as PostgreSQL
    participant IB as inbox.py
    participant AC as AcceptInvoiceDrafts

    U->>S: carrier '/ABC+123' + verification code
    S->>LC: LinkCarrierCommand
    LC->>G: verify credentials
    LC->>DB: INSERT einvoice_carriers (verification encrypted)
    loop every 6 h (jittered) for due carriers, batch of 50
        W->>G: query invoice headers since last_synced_at
        W->>G: query details for new invoice numbers
        W->>W: suggest category (merchant history + rules)
        W->>DB: INSERT einvoices … ON CONFLICT (carrier_id, invoice_number) DO NOTHING
        W->>DB: INSERT notifications 'einvoice_new' (count)
    end
    U->>IB: open Inbox (1 SQL: pending invoices + suggested category names)
    U->>IB: accept selected (batch)
    IB->>AC: AcceptInvoiceDraftsCommand(ids, overrides)
    AC->>DB: INSERT transactions (multi-row, source='einvoice'), UPDATE einvoices SET status='accepted'
```

## 9. Invitation & join

```mermaid
sequenceDiagram
    actor O as Owner (Mei)
    actor I as Invitee (Jason)
    participant F as family.py
    participant CI as CreateInvitation
    participant AI as AcceptInvitation
    participant DB as PostgreSQL
    O->>F: Invite (role = member)
    F->>CI: command
    CI->>DB: INSERT invitations(token_hash = sha256(token), expires_at = now()+7d)
    CI-->>F: URL https://app/…/join?t=<token> and QR (st.image)
    O-->>I: share via LINE / copy link
    I->>F: open link → sign in (st.login) → confirm screen (privacy explainer)
    F->>AI: AcceptInvitationCommand(token)
    AI->>DB: SELECT invitation by token_hash FOR UPDATE; check expiry, plan member limit
    AI->>DB: INSERT family_members; INSERT ledger_members for group's default shared ledger; UPDATE invitations
    AI->>DB: INSERT outbox_events (MemberJoined)
```

## 10. Export & account deletion

- **Export (DM-3):** `ExportLedger` streams rows from a server-side cursor (`yield_per(2000)`) into CSV or XLSX in memory (or a temp file when over 20 MB), then uploads it to S3 under `exports/<user>/<uuid>`. The UI offers `st.download_button` for small files, or a pre-signed URL that is valid for 15 minutes. One query, streamed; no per-row lookups (category, account, and payer names are joined in).
- **Deletion (DM-5):** `RequestAccountDeletion` sets `users.deletion_requested_at`, signs the user out, and removes them from family groups. Their personal-ledger data is scheduled for purge. Their entries in shared ledgers are **kept** so other members' balances stay correct. After 30 days, `PurgeDeletedUsers` (batched) first detaches the user's accounts from shared transactions (sets `account_id` / `account_amount_minor` and `to_account_id` / `to_account_amount_minor` to `NULL` in one `UPDATE … WHERE account_id = ANY(:ids)` each). It then hard-deletes the personal ledger, accounts, carriers, notifications, and auth identities. It then **anonymizes** the `users` row (`display_name = 'Former member'`, `email = NULL`) instead of deleting it, so foreign keys from shared transactions stay valid.
