# 05 — API Design

Kinbook's MVP has **no HTTP API of its own** (see [ADR-002](01-architecture.md#6-architecture-decision-records-summary)). Streamlit pages call the **application-layer API** in-process. That API is the contract between the UI and the core, and the future REST adapter exposes the same contract.

1. [Conventions](#1-conventions)
2. [Application API — commands and queries](#2-application-api)
3. [Outbound integration ports](#3-outbound-integration-ports)
4. [Future REST API](#4-future-rest-api-phase-4)

## 1. Conventions

### 1.1 Shape

```python
# application/use_cases/transactions/record_transaction.py
class RecordTransactionCommand(BaseModel, frozen=True):
    ledger_id: LedgerId
    type: Literal["expense", "income", "transfer"]
    amount_minor: PositiveInt = Field(le=99_999_999_999)
    currency: CurrencyCode                  # validated ISO 4217
    occurred_at: AwareDatetime
    account_id: AccountId | None = None
    to_account_id: AccountId | None = None
    category_id: CategoryId | None = None
    payer_user_id: UserId | None = None     # defaults to actor
    split: SplitSpecDTO | None = None       # required for shared-ledger expense; defaults to ledger default
    note: str | None = Field(default=None, max_length=500)
    merchant: str | None = Field(default=None, max_length=120)
    tag_names: tuple[str, ...] = ()
    fx_rate_override: Decimal | None = None

class RecordTransactionResult(BaseModel, frozen=True):
    transaction_id: TransactionId
    ledger_revision: int

class RecordTransaction:
    def __init__(self, uow_factory: UoWFactory, access: AccessPolicy, fx: FxRateProvider,
                 clock: Clock, ids: IdGenerator) -> None: ...
    def __call__(self, actor: UserId, cmd: RecordTransactionCommand) -> RecordTransactionResult: ...
```

- **Commands** change state. Each runs in exactly one Unit of Work and returns a small result (IDs plus new revision).
- **Queries** never change state. They return frozen DTOs and are safe to cache.
- The first parameter is always `actor: UserId`. Identity never travels inside the command.
- Money crosses the boundary as `amount_minor: int` + `currency: str`. The UI converts to and from display units with `presentation/components/money_input.py`.

### 1.2 Errors

| Exception (application) | Meaning | UI behavior | Future HTTP |
|-------------------------|---------|-------------|-------------|
| `ValidationError` (Pydantic) | Malformed command | Inline field errors | 422 |
| `NotFound` | Entity missing **or not visible to actor** (no existence leak) | `st.error` + navigate back | 404 |
| `AuthorizationError` | Visible but action not permitted (e.g. viewer writes) | `st.warning` | 403 |
| `DomainError` subclasses (`InvalidSplit`, `CurrencyMismatch`, `CategoryKindMismatch`, `LedgerArchived`, `MemberLimitReached`, `InvitationExpired`, …) | Business rule violated | Localized message by `error.code` | 409 / 422 |
| `ConcurrencyConflict(current_version, changed_by)` | Optimistic lock failed | Conflict dialog (Reload / Re-apply) | 409 |
| `ExternalServiceError` | E-invoice / FX / Resend failure | Retry hint; never blocks core logging | 502 |

Every `DomainError` has a stable `code` (e.g. `split.sum_mismatch`) that maps to a key in the i18n bundles.

### 1.3 Authorization matrix (enforced by `AccessPolicy`)

| Action | Personal ledger owner | Shared ledger member (`can_write`) | Shared ledger viewer | Group owner | Other user |
|--------|----------------------|-------------------------------------|----------------------|-------------|------------|
| Read ledger / reports | ✅ | ✅ | ✅ | ✅ if ledger member | ❌ (`NotFound`) |
| Create transaction | ✅ | ✅ | ❌ | ✅ if ledger member | ❌ |
| Edit / delete transaction | ✅ | ✅ own entries; others' entries only if `ledger.allow_edit_others` (default **true** for MVP) | ❌ | ✅ | ❌ |
| Manage categories / budgets | ✅ | ✅ | ❌ | ✅ | ❌ |
| Manage members, delete shared ledger | — | ❌ | ❌ | ✅ | ❌ |
| Read account balance | owner only, or via `account_shares` | via share | via share | via share | ❌ |

## 2. Application API

Column **Q** is the query budget from [06 § 6.3](06-query-performance.md#63-query-budgets-contract). "—" means a single small statement.

### 2.1 Identity & onboarding (F01)

| Name | Kind | Input | Output | Notes |
|------|------|-------|--------|-------|
| `SignInUser` | command | `issuer, subject, email?, name?, locale?` (from `st.user`) | `actor_id, is_new` | First sign-in provisions user, personal ledger, default categories, Cash account |
| `CompleteOnboarding` | command | `default_currency, locale, accounts[] (name, type, opening_balance_minor), monthly_budget_minor?` | — | Idempotent |
| `UpdateProfile` | command | `display_name?, locale?, time_zone?, theme?, week_start?, month_start_day?, reminder_time?` | — | |
| `GetMe` | query | — | `MeDTO (profile, personal_ledger_id, group?, role?, plan)` | Q 1 |

### 2.2 Transactions (F02)

| Name | Kind | Input | Output | Q |
|------|------|-------|--------|---|
| `RecordTransaction` | command | see §1.1 | `transaction_id, ledger_revision` | ≤ 9 |
| `UpdateTransaction` | command | `transaction_id, expected_version, patch fields…` | `version, ledger_revision` | ≤ 9 |
| `DeleteTransaction` | command | `transaction_id, expected_version` | — (soft delete) | ≤ 5 |
| `RestoreTransaction` | command | `transaction_id` | — | ≤ 5 |
| `DuplicateTransaction` | command | `transaction_id, occurred_at?` | `transaction_id` | ≤ 9 |
| `MoveTransactionToLedger` | command | `transaction_id, target_ledger_id, split?` | `transaction_id` | ≤ 9; category re-mapped by slug, else "Other" |
| `AttachReceipt` | command | `transaction_id, filename, content_type, bytes` | `attachment_id` | Uploads to S3, then inserts row |
| `RemoveAttachment` | command | `attachment_id` | — | |
| `CreateRecurringRule` / `UpdateRecurringRule` / `PauseRecurringRule` / `DeleteRecurringRule` | command | `ledger_id, rrule, dtstart, until?, mode, template` | `rule_id` | |
| `ConfirmRecurringOccurrence` | command | `rule_id, occurrence_date, overrides?` | `transaction_id` | For `mode=remind` |
| `RunDueRecurringRules` | command (worker) | `today` | `created_count` | ≤ 8 per batch |
| `GetQuickAddContext` | query | `ledger_id?` | `accounts[], ledgers[], recent_categories[≤8], categories[] (sort order), last_account_id, ledger_pref_by_category{}` | 4 |
| `SuggestCategory` | query | `ledger_id, merchant?, note?` | `category_id?, confidence` | 1 |
| `ListTransactions` | query | `TxFilter (ledger_id, date range, amount range, category_ids, account_ids, tag_names, member_ids, text, types), cursor?` | `TxPage (items[], next_cursor)` | 2 |
| `GetTransactionDetail` | query | `transaction_id` | `TxDetailDTO (fields, splits[], tags[], attachments[] with signed URLs, history[])` | 3 |
| `ListTrash` | query | `ledger_id` | `TxListItem[]` | 1 |

`SplitSpecDTO`:

```python
class SplitSpecDTO(BaseModel, frozen=True):
    method: Literal["equal", "percentage", "exact", "shares"]
    members: tuple[SplitMemberDTO, ...]      # user_id + value (ignored for equal)

class SplitMemberDTO(BaseModel, frozen=True):
    user_id: UserId
    value: Decimal | None = None             # % (sum 100), shares (>0), or exact minor units (sum = amount)
```

### 2.3 Accounts (F03)

| Name | Kind | Input | Output | Q |
|------|------|-------|--------|---|
| `CreateAccount` | command | `name, type, currency, opening_balance_minor, credit_limit_minor?, statement_day?, due_day?` | `account_id` | |
| `UpdateAccount` | command | `account_id, expected_version, fields…` | `version` | |
| `ArchiveAccount` / `UnarchiveAccount` | command | `account_id` | — | |
| `ReorderAccounts` | command | `account_ids[]` (new order) | — | 1 `UPDATE … FROM (VALUES …)` |
| `AdjustAccountBalance` | command | `account_id, actual_balance_minor, occurred_at` | `transaction_id` (type `adjustment`) | |
| `ShareAccount` / `UnshareAccount` (P1) | command | `account_id, permission` | — | Owner only |
| `ListAccountsWithBalances` | query | `include_shared: bool` | `AccountBalanceDTO[]` | 1 |
| `GetNetWorth` (P1) | query | `as_of?` | `assets_minor, liabilities_minor, by_type[]` (in default currency) | 1 |

### 2.4 Family & shared ledgers (F04)

| Name | Kind | Input | Output | Q |
|------|------|-------|--------|---|
| `CreateFamilyGroup` | command | `name` | `group_id, default_ledger_id` | Creates "Household" shared ledger with default categories |
| `RenameFamilyGroup` | command | `group_id, name` | — | Owner |
| `CreateInvitation` | command | `group_id, role` | `invitation_id, url, expires_at` | Owner |
| `RevokeInvitation` | command | `invitation_id` | — | Owner |
| `PreviewInvitation` | query | `token` | `group_name, inviter_name, role, expired` | 1 |
| `AcceptInvitation` | command | `token` | `group_id` | Checks plan member limit |
| `ChangeMemberRole` | command | `group_id, user_id, role` | — | Owner; cannot demote last owner |
| `RemoveMember` / `LeaveGroup` | command | `group_id, user_id` | — | Blocked while the member has a non-zero balance unless `force=true` (balances are kept) |
| `TransferOwnership` | command | `group_id, new_owner_id` | — | |
| `CreateSharedLedger` | command | `group_id, name, currency, member_ids[], default_split_method` | `ledger_id` | |
| `UpdateSharedLedger` | command | `ledger_id, name?, member_ids?, default_split_method?, default_weights?` | — | |
| `ArchiveLedger` | command | `ledger_id` | — | Owner |
| `RecordSettlement` | command | `ledger_id, debtor_id, creditor_id, amount_minor, from_account_id?, to_account_id?, occurred_at` | `transaction_id` | |
| `GetFamilyOverview` | query | `group_id` | `members[], ledgers[], open_invitations[]` | 3 |
| `GetLedgerBalances` | query | `ledger_id` | `nets[] (user, net_minor)`, `suggestions[] (from, to, amount_minor)` | 1 |
| `ListActivity` (P1) | query | `ledger_id, cursor?` | `ActivityItem[]` (from `transaction_history` ⋈ users) | 1 |

### 2.5 Budgets & goals (F05)

| Name | Kind | Input | Output | Q |
|------|------|-------|--------|---|
| `SetBudget` | command | `ledger_id, category_id?, amount_minor, rollover, alert_thresholds[], effective_from` | `budget_id` | Upsert per scope + period |
| `DeleteBudget` | command | `budget_id` | — | |
| `EvaluateBudgetAlerts` | command (worker) | `ledger_ids[]` | `alerts_sent` | 3 per batch |
| `GetBudgetProgress` | query | `ledger_id, period_start` | `BudgetProgressDTO[] (scope, limit, carried_over, spent, pct, status)` | 1 |
| `CreateGoal` / `UpdateGoal` / `DeleteGoal` (P1) | command | `name, target_minor, currency, deadline?, account_id?, ledger_id?` | `goal_id` | |
| `GetGoalProgress` (P1) | query | — | `GoalDTO[] (saved_minor, pct, projected_completion)` | 1 |

### 2.6 Reports (F06)

All report queries take `ReportScope(ledger_id, from, to, kind="expense"|"income")`.

| Name | Output | Q |
|------|--------|---|
| `GetMonthlyOverview` | `income, expense, net` for the period and the previous period, plus `delta_pct` | 1 |
| `GetCategoryBreakdown` | `top_categories[] (id, name, icon, color, total, pct, children[])` | 1 |
| `GetSpendingTrend` | `series[] (month, total)` per selected categories, 6 or 12 months | 1 |
| `GetMemberSpending` (P1) | `members[] (user, paid_total, share_total)` | 1 |
| `GetDailyCalendar` (P1) | `days[] (date, total)` | 1 |
| `SendMonthlySummaries` (worker) | count | Batched per 200 users |

### 2.7 Data management (F07)

| Name | Kind | Input | Output |
|------|------|-------|--------|
| `ExportLedger` | command | `ledger_id, from?, to?, format: csv\|xlsx\|json` | `ExportFile (bytes or signed_url, filename)` |
| `PreviewCsvImport` (P1) | query | `ledger_id, file bytes, preset?, column_mapping?` | `parsed_rows[≤50], errors[], unmapped_categories[]` |
| `CommitCsvImport` (P1) | command | `ledger_id, file_ref, mapping, category_mapping` | `import_batch_id, row_count` |
| `UndoImport` (P1) | command | `import_batch_id` | — (soft-deletes batch rows) |
| `RequestAccountDeletion` | command | `confirm_phrase` | — |
| `CancelAccountDeletion` | command | — | — (within 30 days) |
| `PurgeTrash` / `PurgeDeletedUsers` / `ExpireExports` | command (worker) | — | counts |

### 2.8 Settings, categories, notifications (F08)

| Name | Kind | Input | Output |
|------|------|-------|--------|
| `CreateCategory` | command | `ledger_id, kind, name, parent_id?, icon, color` | `category_id` |
| `UpdateCategory` | command | `category_id, name?, icon?, color?, parent_id?` | — |
| `ReorderCategories` | command | `ledger_id, kind, ordered_ids[]` | — |
| `MergeCategories` | command | `source_id, target_id` | `moved_count` (one `UPDATE transactions SET category_id …`) |
| `ArchiveCategory` | command | `category_id` | — |
| `ListCategories` | query | `ledger_id, kind, include_archived` | `CategoryTreeDTO` (1 query, tree built in memory) |
| `UpdateNotificationSettings` | command | `settings[] (kind, in_app, email)` | — |
| `ListNotifications` | query | `unread_only, cursor?` | `NotificationDTO[]` |
| `CountUnreadNotifications` | query | — | `int` |
| `MarkNotificationsRead` | command | `ids[] \| all` | — |
| `SendLoggingReminders` (worker) | command | `now` | count |

### 2.9 E-invoice (F02 TX-8, P1)

| Name | Kind | Input | Output |
|------|------|-------|--------|
| `LinkCarrier` | command | `carrier_code, verification_code, default_ledger_id?, default_account_id?` | `carrier_id` |
| `UnlinkCarrier` | command | `carrier_id` | — |
| `SyncCarrierInvoices` | command (worker) | `batch_size=50` | `fetched_count` |
| `ListInvoiceInbox` | query | `cursor?` | `InvoiceDraftDTO[] (invoice, items, suggested_category)` |
| `AcceptInvoiceDrafts` | command | `items[] (einvoice_id, category_id?, ledger_id?, account_id?, split?)` | `transaction_ids[]` |
| `IgnoreInvoiceDrafts` | command | `einvoice_ids[]` | — |

## 3. Outbound integration ports

Defined in `application/ports/gateways.py` as `typing.Protocol`s. The infrastructure adapters implement them, and tests use fakes.

```python
class EInvoiceGateway(Protocol):
    def verify_carrier(self, carrier_code: str, verification_code: str) -> bool: ...
    def list_headers(self, carrier_code: str, verification_code: str,
                     start: date, end: date) -> list[InvoiceHeader]: ...
    def get_details(self, carrier_code: str, verification_code: str,
                    headers: Sequence[InvoiceHeader]) -> list[InvoiceDetail]: ...

class FxRateProvider(Protocol):
    def rate(self, base: CurrencyCode, quote: CurrencyCode, on: date) -> Decimal: ...   # reads fx_rates
    def refresh(self, on: date) -> int: ...                                             # worker only

class BlobStorage(Protocol):
    def put(self, key: str, data: bytes, content_type: str) -> None: ...
    def signed_url(self, key: str, ttl_seconds: int = 900) -> str: ...
    def delete_many(self, keys: Sequence[str]) -> None: ...

class EmailSender(Protocol):
    def send_many(self, messages: Sequence[EmailMessage]) -> None: ...

class Clock(Protocol):
    def now(self) -> datetime: ...          # timezone-aware UTC

class IdGenerator(Protocol):
    def new(self) -> UUID: ...              # UUIDv7
```

| Adapter | External system | Notes |
|---------|-----------------|-------|
| `MofEInvoiceGateway` | Taiwan Ministry of Finance e-invoice platform API (carrier invoice header and detail queries) | Requires an AppID and API key issued by MOF. HTTPS with signed parameters. Endpoint names and signing must be confirmed against the current MOF API specification before implementation. Rate-limited: max 1 request/s per carrier, exponential backoff. |
| `BotFxRateProvider` | Bank of Taiwan published daily rates (CSV) | Fallback: a commercial FX API. Stored in `fx_rates`; the app never calls the provider during a user request. |
| `S3BlobStorage` | S3 / MinIO | Private bucket; SSE enabled; keys `receipts/<ledger>/<uuid>`, `exports/<user>/<uuid>` |
| `ResendEmailSender` | Resend email API (`resend` Python SDK) | Templates rendered with Jinja2, localized. `send_many` uses Resend batch sending in chunks of 100 and retries with backoff on 429/5xx. `ConsoleEmailSender` (logs only) is used in `local`. |

## 4. Future REST API (Phase 4)

When native mobile clients are added, a thin **FastAPI** adapter (`src/kinbook/presentation/rest/`) maps HTTP to the same use cases and queries. No domain or application code changes. Auth: OIDC bearer tokens from the same broker.

| Method & path | Maps to |
|---------------|---------|
| `GET /v1/me` | `GetMe` |
| `GET /v1/ledgers/{id}/transactions?cursor=&from=&to=&q=` | `ListTransactions` |
| `POST /v1/ledgers/{id}/transactions` | `RecordTransaction` |
| `GET /v1/transactions/{id}` | `GetTransactionDetail` |
| `PATCH /v1/transactions/{id}` (header `If-Match: <version>`) | `UpdateTransaction` |
| `DELETE /v1/transactions/{id}` (`If-Match`) | `DeleteTransaction` |
| `POST /v1/transactions/{id}/attachments` | `AttachReceipt` |
| `GET /v1/accounts` | `ListAccountsWithBalances` |
| `POST /v1/accounts` · `PATCH /v1/accounts/{id}` | `CreateAccount` · `UpdateAccount` |
| `GET /v1/ledgers/{id}/balances` | `GetLedgerBalances` |
| `POST /v1/ledgers/{id}/settlements` | `RecordSettlement` |
| `GET /v1/ledgers/{id}/budgets?period=` · `PUT /v1/ledgers/{id}/budgets` | `GetBudgetProgress` · `SetBudget` |
| `GET /v1/ledgers/{id}/reports/{overview\|categories\|trend}` | report queries |
| `POST /v1/groups` · `POST /v1/groups/{id}/invitations` · `POST /v1/invitations/{token}/accept` | family commands |
| `GET /v1/sync?since_revision=` | New query for offline-capable clients: changed rows per ledger since a revision |

Conventions for that phase: JSON with `amount_minor` integers, RFC 7807 problem details for errors (`type` = domain error code), cursor pagination, an `ETag` derived from `version`, and `Idempotency-Key` headers on POST (stored for 24 h).
