# Kinbook Implementation Plan — Dependency-Ordered Roadmap + M1 Detailed Tasks

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the merged design docs into a dependency-safe build order (tech foundations → features), with M1 (walking skeleton) specified task-by-task so implementation can start immediately.

**Architecture:** Clean architecture (domain → application → infrastructure → presentation) on Python 3.14 + Streamlit, SQLite in local dev and PostgreSQL 16 in CI/staging/prod, worker process for async jobs, outbox for notifications. Every milestone ends with working, testable software.

**Tech Stack:** Python 3.14, uv, Streamlit ≥ 1.50, SQLAlchemy 2.0, Alembic, Pydantic v2, psycopg 3, APScheduler 3, Resend, S3/MinIO, Keycloak (OIDC broker), pytest + Hypothesis + testcontainers + AppTest + Playwright, Ruff, mypy, import-linter, Semgrep.

**Spec:** `doc/plan/accounting-app-prd.md`, `doc/features/README.md` + `F01`–`F09`, `doc/tech/01`–`06` (repo `C:\Users\Kuei.Chen\appium_git\accounting`, merged in PR #1).

## Context

The repo now holds a PRD, a 9-feature breakdown, and 6 technical design docs, but no code. The user asked for an implementation priority that respects **feature ↔ feature** and **feature ↔ tech-foundation** dependencies. The milestone list in `doc/features/README.md` is a sketch. This plan checks it against the actual dependencies in the docs, fixes the ordering problems it finds (see "Ordering corrections"), and adds the external long-lead items. It is too large for one detailed plan (the writing-plans scope rule applies), so:
- **Part A:** the dependency analysis and prioritized roadmap for M0–M7.
- **Part B:** M1 broken into executable TDD tasks.
- Each later milestone gets its own detailed plan when it starts, saved as `doc/plan/milestones/M<n>-<name>.md`.

## Global Constraints

- Python `>=3.14` (`.python-version` = `3.14`); **uv only**: no `pip`, no `requirements.txt`; `uv.lock` committed.
- Dev DB: SQLite `var/kinbook-dev.db` (WAL). CI/staging/prod: PostgreSQL 16. **Every SQL-touching test runs on PostgreSQL** (testcontainers).
- Email: Resend (`resend` SDK) behind the `EmailSender` port; `ConsoleEmailSender` in `local`.
- Layer rule enforced by import-linter: `domain` imports stdlib only; `application` imports no `sqlalchemy`/`streamlit`; `presentation` imports no `infrastructure` except `container.py`.
- Every ORM `relationship()` uses `lazy="raise"`. Every query service and use case has a query-budget test that runs with 1 and 200 rows (budgets in `doc/tech/06` §6.3).
- Money is `int` minor units + ISO 4217 code; `amount_minor > 0`; never float.
- Primary keys: `uuid.uuid7()` (stdlib).
- Widget keys follow `<page>.<element>`; all UI strings live in `i18n/en.toml` + `i18n/zh_TW.toml`.
- Code comments and code strings in English; all `.md` files in English.

## Review Focus

These failure modes are implied by the spec but not exercised by the per-feature tests. Each gets a pinned test in the task that owns the code:
1. **Local-date boundary.** An expense at 00:30 Asia/Taipei on Dec 1 (16:30Z Nov 30) must have `occurred_on = 2026-12-01` and count in December. Test: `test_occurred_on_uses_actor_timezone_across_midnight` in M2-T3.
2. **Shared-cache leakage.** `st.cache_data` is process-wide, so two actors with different visibility must never receive each other's cached DTO. Test: `test_cached_query_key_includes_actor` in M1-T7 (helper), re-asserted for the dashboard in M4-T4.
3. **Double-tap Save** on a slow mobile connection must create one transaction, not two. Test: AppTest `test_quick_add_double_click_records_once` (per-form nonce in session state) in M2-T4.
4. **Amount input for TWD.** `"1,234"` → `123400` minor units. `"12.5"`, `"-5"`, `"0"`, and values over `99_999_999_999` are rejected with a field error. Test: `test_parse_money_input_twd_cases` in M2-T4.
5. **Editing an old transaction whose category or account is now archived** must still save if those fields are unchanged; picking an archived one for a *new* value is rejected. Test: `test_update_keeps_archived_category_when_unchanged` in M2-T5.

---

# Part A — Dependency analysis & priority

## A1. Tech foundations (T) and what needs them

| ID | Foundation | Doc source | Needed first by |
|----|-----------|-----------|-----------------|
| T0 | uv project, Ruff, mypy, import-linter, Semgrep, pre-commit, CI (GitHub Actions + PG service) | tech/01 §2, §2.2; tech/02 §9 | everything |
| T1 | Domain kernel: typed IDs, `Money`/`Currency`, `DomainError` taxonomy | tech/01 §3.2; tech/05 §1.2 | F01 |
| T2 | Persistence: settings, engine (SQLite/PG), `dialect.py` portable types, ORM `Base`, UoW, Alembic baseline, query-count fixtures + runtime guard | tech/01 §2.1; tech/04 §1, §6; tech/06 §3, §6 | F01 |
| T3 | Application kernel: ports (`Clock`, `IdGenerator`, `UnitOfWork`), `AccessPolicy` (personal first) | tech/05 §1.3 | F01 |
| T4 | Presentation shell: `st.login` guard + `auth.py` seam, container, `state.py`, i18n, navigation, idle timeout, `cached_query` with revision key | tech/02 §2–4 | F01 |
| T5 | Worker shell: APScheduler, advisory lock (PG) / no-op (SQLite), job registry | tech/02 §5 | F02-6 recurring, PurgeTrash |
| T6 | Outbox + `DispatchOutbox` + notifications table + bell fragment + `EmailSender` (Console/Resend) | tech/02 §6; tech/03 §6 | F04-9, F05-3, F08-6/7 |
| T7 | `BlobStorage` (local disk dev, S3 prod) + upload validation | tech/02 §7; tech/05 §3 | F02-3 receipts, F07-3 export |
| T8 | FX rates table + `RefreshFxRates` job + `FxRateProvider` | tech/05 §3 | F02-7 multi-currency, F03-5 net worth |
| T9 | Deploy: Dockerfiles, compose, proxy with WebSocket + sticky sessions, staging | tech/02 §1 | beta |
| T10 | Observability: structlog, Sentry, metrics | tech/02 §8 | beta |
| T11 | E2E: Playwright + Locust | tech/02 §9 | beta |

## A2. Feature dependency matrix

| Feature slice | Requires features | Requires tech |
|---------------|-------------------|---------------|
| F01 sign-in + provisioning (incl. default categories + Cash account) | — | T0–T4 |
| F03-1 account CRUD | F01 | T0–T4 |
| F08-1/2 category management | F01 (seeded categories) | T0–T4 |
| F02 core (record, list, search, edit, trash) | F01, F03-1, F08-1 | T2 revision + history |
| F03-2/3 balances + adjustment | **F02 core** (transactions table) | — |
| F02-3 receipts | F02 core | T7 |
| F02-6 recurring | F02 core | T5 |
| F01-4 onboarding wizard (accounts step) | F03-1 | — |
| F08-7/8 notification center | F01 | T5, T6 |
| F04 groups / invites / shared ledgers | F01, F02 core | T6 (invite-accepted notice) |
| F04-7 splits, F04-8 balances & settlement | F04 shared ledgers, F02 core | — |
| F04-9 shared notifications | F04, F08-7 | T6 |
| F05-1/2 budgets + progress | F02 core, F08-1 | — |
| F05-3 alerts | F05-1, F08-7 | T5, T6 |
| F01-4 onboarding budget step | F05-1 | — |
| F06-1..3 reports | F02 core | — |
| Home dashboard | F02, F05-1, F06-1 | T4 caching |
| F07-3 export | F02 core, F04 (masking) | T7 |
| F07-6 account deletion + purge | **F04** (shared-ledger re-attribution) | T5 |
| F01-5 demo mode | all P0 pages exist | fake container |
| F08-6 logging reminder | F02, F08-7 | T5, T6 |
| F02-11 smart category | F02 core | — |
| F09 e-invoice | **F02-11** (suggestion), F04 (accept into shared ledger), F08-7 | T5, T6, MOF AppID |
| F02-7 multi-currency, F03-5 net worth | F02, F03-2 | T8 |
| F07-4/5 CSV import + undo | F02, F08-1 | T5, T7 |

## A3. Ordering corrections to `doc/features/README.md`

1. **Account balances (F03-2) move after F02 core.** They are derived from `transactions`, so in M2 the order is accounts CRUD → transactions → balances.
2. **The onboarding wizard splits in two.** The accounts step lands in M2; the budget step lands in M4, after F05-1.
3. **Notification infrastructure (T6) is the first task of M3.** F04-9, F05-3, and F08-6 all depend on it.
4. **Smart category (F02-11) must precede F09.** F09-4 reuses its history query.
5. **Account deletion (F07-6) needs F04 done**, because shared-ledger rows must stay attributed to "Former member". It stays in M5.
6. **Free-plan limits (`MemberLimitReached`, `LedgerLimitReached`) are enforced in M3**, even though payment integration comes later.
7. **External long-lead items start in M0, in parallel with M1.**

## A4. Critical path

```
T0 → T1/T2/T3 → T4 (auth) → F01 → F03-1 → F02 core → F04 shared+splits → F04 balances
   → T6 notifications → F05 alerts → F07-6 deletion → T9/T11 beta
```

## A5. Prioritized milestones

**M0 — External prerequisites (owner: user; starts now, runs in parallel with M1)**
- Apply for an MOF e-invoice AppID + API key (longest lead; blocks F09).
- OIDC broker: Keycloak hosting or an Auth0 tenant. Register Google OAuth client, Apple Services ID (needs an Apple Developer account), and LINE Login channel.
- Resend: verify the sending domain (SPF/DKIM DNS) and create the API key.
- Staging hosting target: container platform + managed PostgreSQL 16 + S3 bucket.

**M1 — Walking skeleton (P0)** — detailed in Part B.
Exit: sign in → home shows name and personal ledger. CI green (ruff, mypy, import-linter, Semgrep, unit, PG integration with query-budget harness).

**M2 — Personal bookkeeping (P0)**

| Task | Scope (feature IDs) | Depends on |
|------|---------------------|------------|
| M2-T1 | Accounts CRUD, archive, reorder (F03-1) | M1 |
| M2-T2 | Category management + merge (F08-1, F08-2), `ListCategories` | M1 |
| M2-T3 | Transaction aggregate + `RecordTransaction` (personal; expense/income/transfer), ledger revision bump, `transaction_history`, `quick_add_preferences` (F02-1/2/3 minus receipts) | T1, T2 |
| M2-T4 | Quick-add page + `GetQuickAddContext` (Q 4), `money_input`/`parse_money`, form nonce | T3 |
| M2-T5 | `ListTransactions` (Q 2) + trigram search + filters, detail dialog (Q 3), edit with optimistic lock, duplicate, delete, trash, restore (F02-5, F02-12) | T3 |
| M2-T6 | `ListAccountsWithBalances` (Q 1), `AdjustAccountBalance` (F03-2, F03-3) | T3 |
| M2-T7 | `BlobStorage` (T7) + receipts (F02-3 attachments) | T3 |
| M2-T8 | Worker shell (T5) + recurring rules + `RunDueRecurringRules` + `PurgeTrash` (F02-6) | T3 |
| M2-T9 | Profile settings (F08-3, F08-4, reminder time stored) + onboarding wizard steps 1–2 (F01-4) | T1, T2 |

Exit: one user can run a month of personal bookkeeping on SQLite (dev) and PG (CI).

**M3 — Family (P0)**

| Task | Scope | Depends on |
|------|-------|------------|
| M3-T1 | Outbox + `DispatchOutbox` + notifications + bell fragment + notification settings (in-app) + Console/Resend senders (T6; F08-7, F08-8 in-app) | M2-T8 |
| M3-T2 | Family group, invitations (token hash, QR, LINE share), join page with privacy explainer, roles, ownership transfer (F04-1..4) | M3-T1 |
| M3-T3 | Shared ledgers + `ledger_members` + `AccessPolicy` shared rules + full authz matrix tests + free-plan limits (F04-5) | M3-T2 |
| M3-T4 | `SplitAllocator` (largest remainder, payer-first ties) + splits in record/update + split editor UI + ledger choice per category (F04-6, F04-7) | M3-T3 |
| M3-T5 | `GetLedgerBalances` (Q 1) + `BalanceCalculator.suggest_settlements` + `RecordSettlement` (F04-8) | M3-T4 |
| M3-T6 | Shared-ledger notification fan-out + conflict dialog (F04-9 in-app, F04-10) | M3-T1, M3-T4 |
| M3-T7 | `MoveTransactionToLedger` + private account-name masking in lists (F02 rule 7, F03 rule 2) | M3-T4 |

Exit: two users share a ledger, see each other's entries within 60 s, and settle to 0 / 0.

**M4 — Insight (P0)**

| Task | Scope | Depends on |
|------|-------|------------|
| M4-T1 | `SetBudget`/`DeleteBudget`/`GetBudgetProgress` (Q 1) + budgets page + onboarding step 3 (F05-1, F05-2, F01-4) | M2 |
| M4-T2 | `EvaluateBudgetAlerts` from outbox + `budget_alert_log` + email (F05-3) | M4-T1, M3-T1 |
| M4-T3 | Reports overview / breakdown / trend + Plotly theme (F06-1..3) | M2 |
| M4-T4 | Home dashboard (`DashboardQueries` Q 3 + revision) with cache | M4-T1, M4-T3 |
| M4-T5 | Daily logging reminder (F08-6) | M3-T1 |

**M5 — Hardening → closed beta (P0)**

| Task | Scope | Depends on |
|------|-------|------------|
| M5-T1 | Export CSV/XLSX (+ JSON full export, F07-7) streamed with masking (F07-3) | M3-T7, M2-T7 |
| M5-T2 | Account deletion request/cancel + `PurgeDeletedUsers` + `ExpireExports` (F07-6) | M3 |
| M5-T3 | Demo mode with fake container (F01-5) | M4 |
| M5-T4 | Security: upload sniffing/EXIF strip, rate limits, security log events, observability (T10) | M4 |
| M5-T5 | Dockerfiles, proxy (WSS + sticky), staging deploy (T9) | M4 |
| M5-T6 | Playwright golden paths + Locust (200 sessions, p95 < 800 ms) (T11) | M5-T5 |

Exit: ≥ 200 beta households can be onboarded on staging.

**M6 — v1.x (P1)**, ordered by user value, then by dependency:
1. F02-11 smart category → 2. F09 e-invoice (needs the M0 AppID) → 3. F02-8 home-screen shortcut guide → 4. T8 FX + F02-7 multi-currency → 5. F03-5 net worth → 6. Email notifications (F08-8 email) + F06-6 monthly summary → 7. F03-4 card reminders, F05-4 rollover, F05-5 goals → 8. F06-4 member spending, F06-5 calendar, F08-5 week/month start → 9. F07-4/5 CSV import + undo → 10. F03-6 account sharing, `ListActivity`, F02-4 location, F01-7 LINE login / F01-8 passkey (broker configuration), PostgreSQL RLS.

**M7 — P2:** F02-9 OCR, F02-10 voice, F04-11 allowance, F04-12 guests, F05-6 custom periods, F06-7 year in review, REST adapter (tech/05 §4).

## A6. Doc gaps to close while implementing (update the owning doc in the same PR)

- Double-submit protection (Review Focus 3) → add to `doc/features/F02-transactions.md` §4.
- AppTest cannot drive real OIDC → document the `presentation/auth.py` seam in `doc/tech/02` §9.
- `uv run kinbook-seed` (tech/01 §2.1) is not defined anywhere → add as `[project.scripts]` in M1-T5.
- Python 3.14 wheel availability → gate in M1-T1. If it fails, stop and ask the user (fallback 3.13 is their call).

---

# Part B — M1 Walking Skeleton (detailed)

Repo: `C:\Users\Kuei.Chen\appium_git\accounting`. Work on branch `feat/m1-walking-skeleton`. PR + merge at the end (user-authorized workflow).

### Task 1: Project scaffold & tooling

**Files:**
- Create: `pyproject.toml`, `.python-version`, `uv.lock`, `src/kinbook/__init__.py`, `src/kinbook/{domain,application,infrastructure,presentation}/__init__.py`, `.pre-commit-config.yaml`, `.semgrep/no-repo-get-in-loop.yml`, `tests/unit/test_package.py`
- Modify: `.gitignore` (add `var/`, `.venv/`, `.streamlit/secrets.toml`)

**Interfaces:** Produces package `kinbook` (src layout), dependency group `dev`, and import-linter contracts named `layers`, `domain-is-pure`, `application-no-frameworks`, `presentation-no-infrastructure`.

- [ ] **Step 1: Wheel gate.** `uv init --package --python 3.14`, then `uv add streamlit sqlalchemy "psycopg[binary]" pydantic pydantic-settings alembic apscheduler resend boto3 pandas openpyxl plotly structlog` and `uv add --dev pytest pytest-cov hypothesis "testcontainers[postgres]" ruff mypy import-linter pre-commit semgrep`. Expected: resolution succeeds on 3.14. If any package has no 3.14 wheel, STOP and report to the user.
- [ ] **Step 2: Failing test** `tests/unit/test_package.py::test_version` asserts `kinbook.__version__ == "0.1.0"`. Run `uv run pytest tests/unit -q`. Expected: FAIL (attribute missing).
- [ ] **Step 3: Implement.** Set `__version__ = "0.1.0"`. Configure `[tool.ruff]` (line length 100, rules `E,F,I,B,UP,SIM,RUF`), `[tool.mypy] strict = true`, `[tool.importlinter]` with the four contracts (forbidden: `sqlalchemy`, `streamlit`, `pydantic` from domain; `sqlalchemy`, `streamlit` from application; `kinbook.infrastructure` from `kinbook.presentation` with `ignore_imports = kinbook.presentation.container -> kinbook.infrastructure.*`). Add the Semgrep rule flagging `$REPO.get(...)` inside `for`/comprehension bodies.
- [ ] **Step 4: Verify.** `uv run pytest -q && uv run ruff check && uv run ruff format --check && uv run mypy src && uv run lint-imports && uv run semgrep --config .semgrep --error src`. Expected: all pass.
- [ ] **Step 5: Commit** `chore: scaffold uv project with lint, type, and architecture contracts`.

### Task 2: CI workflow

**Files:** Create `.github/workflows/ci.yml`

**Interfaces:** Jobs `lint` (ruff, mypy, lint-imports, semgrep), `unit` (`pytest tests/unit tests/ui`), `integration` (`pytest tests/integration` with a `postgres:16` service and `KINBOOK_TEST_DATABASE_URL`), and aggregator `all-checks-passed` (needs all three). Uses `astral-sh/setup-uv` with `enable-cache: true` and `uv sync --frozen`.

- [ ] **Step 1:** Write the workflow. Pin action versions by tag.
- [ ] **Step 2:** Push the branch. Run `gh pr create --draft`, then `gh pr checks --watch`. Expected: all jobs green. An empty integration suite must pass: use `pytest --suppress-no-test-exit-code` or add the placeholder from Task 4.
- [ ] **Step 3: Commit** `ci: add lint, unit, and PostgreSQL integration jobs`.

### Task 3: Domain kernel

**Files:** Create `src/kinbook/domain/ids.py`, `money.py`, `errors.py`; tests `tests/unit/domain/test_money.py`, `test_errors.py`

**Interfaces (Produces):**
- `ids.py`: `UserId`, `LedgerId`, `AccountId`, `CategoryId`, `TransactionId` = `NewType(..., UUID)`; `new_id() -> UUID` (wraps `uuid.uuid7()`).
- `money.py`: `@dataclass(frozen=True) Currency(code: str, exponent: int)`; `get_currency(code: str) -> Currency` (table: TWD 2, USD 2, JPY 0, EUR 2, … raises `UnknownCurrency`); `@dataclass(frozen=True) Money(amount_minor: int, currency: Currency)` with `__add__`/`__sub__` (raise `CurrencyMismatch`).
- `errors.py`: `class DomainError(Exception)` with `code: ClassVar[str]`; subclasses `UnknownCurrency("currency.unknown")`, `CurrencyMismatch("currency.mismatch")`, `NotFound("not_found")`, `AuthorizationError("forbidden")`.

- [ ] **Step 1: Failing tests:** `test_get_currency_twd_exponent_2` (`get_currency("TWD").exponent == 2`), `test_get_currency_unknown_raises` (`"XXX"` → `UnknownCurrency`, `.code == "currency.unknown"`), `test_money_add_same_currency` (`Money(100,TWD)+Money(50,TWD) == Money(150,TWD)`), `test_money_add_mismatch_raises`, `test_new_id_is_uuid7` (`new_id().version == 7`), `test_new_ids_are_time_ordered` (1,000 ids are already sorted).
- [ ] **Step 2:** `uv run pytest tests/unit/domain -q`. Expected: FAIL (import errors).
- [ ] **Step 3:** Implement per Interfaces.
- [ ] **Step 4:** Rerun. Expected: PASS. Also `uv run lint-imports` passes.
- [ ] **Step 5: Commit** `feat(domain): add ids, money, and error kernel`.

### Task 4: Persistence foundation

**Files:** Create `src/kinbook/infrastructure/config.py`, `db/engine.py`, `db/dialect.py`, `db/orm_models.py`, `db/uow.py`, `application/ports/uow.py`, `application/ports/gateways.py` (`Clock`, `IdGenerator` only for now), `alembic.ini`, `migrations/env.py`, `migrations/versions/0001_baseline.py`, `tests/integration/conftest.py`, `tests/integration/test_migrations.py`, `tests/unit/infrastructure/test_dialect.py`, `tests/unit/infrastructure/test_orm_config.py`

**Interfaces (Produces):**
- `Settings(BaseSettings)` with env prefix `KINBOOK_`: `env: Literal["local","ci","staging","prod"] = "local"`, `database_url: str = "sqlite:///var/kinbook-dev.db"`, `resend_api_key: SecretStr | None = None`, `email_from: str = "Kinbook <noreply@example.com>"`, `idle_timeout_minutes: int = 30`.
- `create_engine_from_settings(s: Settings) -> Engine`. SQLite: `PRAGMA journal_mode=WAL; foreign_keys=ON`, `BEGIN IMMEDIATE` on write transactions. PostgreSQL: `pool_size=5, max_overflow=5, pool_pre_ping=True`, plus connect options `statement_timeout=5000, lock_timeout=2000, idle_in_transaction_session_timeout=10000`.
- `dialect.py`: `UTCDateTime` (TypeDecorator; rejects naive datetimes; returns aware UTC), `DecimalAsText`, `JSONType = JSON().with_variant(JSONB, "postgresql")`.
- `orm_models.Base(DeclarativeBase)`.
- `UnitOfWork` Protocol (`__enter__`, `__exit__`, `commit()`, `rollback()`); `SqlUnitOfWork(session_factory)`; `UoWFactory = Callable[[], UnitOfWork]`.
- Test fixtures `pg_engine` (testcontainers PostgreSQL 16, Alembic upgraded once per session) and `assert_max_queries(n)` (as in tech/06 §6.1). Runtime guard `install_query_counter(engine)` + `queries_in_current_rerun() -> int` via `contextvars`.

- [ ] **Step 1: Failing tests:** `test_utc_datetime_rejects_naive`, `test_utc_datetime_roundtrip_sqlite` (aware in → aware UTC out), `test_decimal_as_text_preserves_8_places` (`Decimal("31.12345678")` round-trips exactly on SQLite), `test_all_relationships_raise_on_lazy_load` (tech/06 §3), `test_migrations_upgrade_downgrade_postgres` (upgrade head → downgrade base → upgrade head on `pg_engine`), `test_migrations_upgrade_sqlite`, `test_assert_max_queries_fails_when_exceeded` (2 `SELECT 1` under budget 1 → `AssertionError`).
- [ ] **Step 2:** Run unit + integration. Expected: FAIL.
- [ ] **Step 3:** Implement. Alembic `env.py` uses `render_as_batch=True` when the dialect is sqlite; `0001_baseline` creates the PG extensions (`pg_trgm`, `citext`) only when the dialect is postgresql.
- [ ] **Step 4:** Rerun both suites. Expected: PASS.
- [ ] **Step 5: Commit** `feat(infra): add settings, engines, portable types, UoW, migrations, and query-budget harness`.

### Task 5: Identity schema, default seeds, `SignInUser`

**Files:** Create `domain/model/user.py`, `domain/model/ledger.py`, `domain/model/category.py`, `domain/model/account.py`, `application/seeds/default_categories.py`, `application/ports/repositories.py`, `application/use_cases/identity/sign_in_user.py`, `infrastructure/db/repositories/users.py`, `infrastructure/db/mappers.py`, `migrations/versions/0002_identity.py`, `src/kinbook/infrastructure/cli/seed.py`; tests `tests/unit/application/test_sign_in_user.py` (in-memory fakes in `tests/fakes.py`), `tests/integration/test_sign_in_user_pg.py`
- Modify: `orm_models.py`; `pyproject.toml` (`[project.scripts] kinbook-seed = "kinbook.infrastructure.cli.seed:main"`)

**Interfaces (Produces):**
- Tables (DDL exactly per tech/04 §3.1–3.2): `users`, `auth_identities`, `ledgers`, `categories`, `accounts`.
- `DEFAULT_EXPENSE_CATEGORIES` / `DEFAULT_INCOME_CATEGORIES`: tuples of `SeedCategory(slug, name_en, name_zh, icon, color, children)` copied from PRD §14.1/14.2 (slugs like `food.lunch`, `family.red_envelopes`).
- `SignInCommand(issuer: str, subject: str, email: str | None, email_verified: bool, name: str | None, locale: str | None)`; `SignInResult(actor_id: UserId, personal_ledger_id: LedgerId, is_new: bool)`.
- `SignInUser(uow_factory: UoWFactory, clock: Clock, ids: IdGenerator).__call__(cmd: SignInCommand) -> SignInResult`.
- `UserRepository.find_by_identity(issuer, subject) -> UserId | None`, `add_provisioned(bundle: ProvisionedUser) -> None` (users + identity + personal ledger + categories as one multi-row insert + Cash account).

- [ ] **Step 1: Failing unit tests:**
  - `test_first_sign_in_provisions_bundle`: `is_new=True`; 1 personal ledger; categories count == the seed total; 1 account named "Cash" of type `cash`.
  - `test_second_sign_in_is_idempotent`: same `actor_id`, `is_new=False`, nothing new created.
  - `test_locale_defaults`: `"zh-TW"` → currency TWD; `"en"` → USD; `None` → zh-TW/TWD.
  - `test_unverified_email_not_stored`.
- [ ] **Step 2: Failing integration tests:** `test_returning_sign_in_query_budget` (`assert_max_queries(2)`); `test_provisioning_is_atomic` (fake seed raises mid-way → zero rows in `users`); `test_cash_account_currency_matches_default`.
- [ ] **Step 3:** Run. Expected: FAIL.
- [ ] **Step 4:** Implement. `kinbook-seed` creates a demo user (`issuer="local"`, `subject="dev"`) with sample data via `SignInUser` (SQLite dev only).
- [ ] **Step 5:** Run unit + integration. Expected: PASS.
- [ ] **Step 6: Commit** `feat(identity): provision users with personal ledger, default categories, and cash account`.

### Task 6: `AccessPolicy` (personal) + `GetMe`

**Files:** Create `application/authz.py`, `application/queries/me.py` (Protocol + `MeDTO`), `infrastructure/db/query_services/me.py`; tests `tests/unit/application/test_authz_personal.py`, `tests/integration/test_get_me_pg.py`

**Interfaces (Produces):**
- `AccessPolicy(lookup: LedgerAccessLookup)` with `require_ledger_read(actor: UserId, ledger_id: LedgerId) -> None` and `require_ledger_write(...)`. Raises `NotFound` for invisible ledgers and `AuthorizationError` for visible but read-only ones. In M1 only the personal-owner rule exists; M3 adds the shared rules.
- `MeQueries.get(actor: UserId) -> MeDTO(user_id, display_name, locale, time_zone, default_currency, theme, personal_ledger_id, onboarded: bool)`.

- [ ] **Step 1: Failing tests:** `test_owner_can_read_and_write_personal_ledger`; `test_other_user_gets_not_found` (raises `NotFound`, not `AuthorizationError`); `test_get_me_query_budget_is_1`.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. `GetMe` is a single `users ⋈ ledgers(kind='personal')` select.
- [ ] **Step 4:** Run. Expected: PASS.
- [ ] **Step 5: Commit** `feat(authz): add personal-ledger access policy and GetMe query`.

### Task 7: Presentation shell (auth, container, state, i18n, caching, idle timeout)

**Files:** Create `presentation/streamlit_app.py`, `presentation/auth.py`, `presentation/container.py`, `presentation/state.py`, `presentation/caching.py`, `presentation/i18n/loader.py`, `presentation/i18n/en.toml`, `presentation/i18n/zh_TW.toml`, `presentation/pages/home.py`, `presentation/pages/sign_in.py`, `.streamlit/config.toml`, `.streamlit/secrets.example.toml`, `docker/compose.yaml` (profiles: `keycloak` default for local sign-in; `postgres` adds PG 16 + MinIO), `docker/keycloak/realm-kinbook.json` (test user `dev@kinbook.local`); tests `tests/ui/test_shell.py`, `tests/unit/presentation/test_idle.py`, `tests/unit/presentation/test_caching.py`, `tests/unit/presentation/test_i18n.py`

**Interfaces (Produces):**
- `auth.current_identity() -> Identity | None` (the only place reading `st.user`); `auth.login() -> None` (`st.login("kinbook")`); `auth.logout() -> None`.
- `container.get_container() -> Container` (`@st.cache_resource`). Fields: `settings`, `engine`, `sign_in_user`, `me_queries`, `access`, `ledger_revisions`.
- `state.py`: typed accessors `get_actor_id() -> UserId | None`, `set_actor_id(UserId)`, `touch_activity(now)`, `get_last_activity() -> datetime | None`.
- `is_idle_expired(last_activity: datetime | None, now: datetime, timeout: timedelta) -> bool`.
- `caching.cached_query(name: str, actor: UserId, ledger_id: LedgerId, fn: Callable[..., T], *params) -> T`. It authorizes via `access.require_ledger_read` **before** the cache, reads the revision, and calls an `@st.cache_data(ttl=600, max_entries=5000)` inner function keyed `(name, actor, ledger_id, revision, params)`.
- `i18n.t(key: str, locale: str, **fmt) -> str`. A missing key raises in `local`/`ci` and falls back to `en` in prod.
- Widget keys: `sign_in.login`, `home.greeting`.

- [ ] **Step 1: Failing tests:**
  - `test_idle_expired_after_30_minutes`: `last = now - 31 min` → True; `now - 29 min` → False; `None` → False.
  - `test_cached_query_key_includes_actor` (Review Focus 2): two actors on the same ledger and revision get separate cache entries; the inner function is called twice.
  - `test_cached_query_authorizes_before_cache`: an unauthorized actor raises `NotFound` even when the cache is warm.
  - `test_i18n_bundles_have_same_keys`: en and zh_TW have identical key sets.
  - AppTest `test_signed_out_shows_only_sign_in`: patch `current_identity → None`; `sign_in.login` exists and no home elements render.
  - AppTest `test_signed_in_home_shows_display_name`: patch the identity and a fake container; `home.greeting` contains the name.
- [ ] **Step 2:** `uv run pytest tests/unit/presentation tests/ui -q`. Expected: FAIL.
- [ ] **Step 3:** Implement. The `streamlit_app.py` bootstrap order is: container → identity guard (`st.stop()`) → `SignInUser` once per session (actor cached in state) → idle check → `st.navigation([home])`.
- [ ] **Step 4:** Rerun. Expected: PASS.
- [ ] **Step 5: Manual smoke.** `docker compose -f docker/compose.yaml up -d keycloak`, copy `secrets.example.toml` to `secrets.toml`, then `uv run alembic upgrade head && uv run streamlit run src/kinbook/presentation/streamlit_app.py`. Sign in as `dev@kinbook.local`. Expected: the home page greets the user. `var/kinbook-dev.db` has 1 user, 1 personal ledger, the seeded categories, and a Cash account.
- [ ] **Step 6: Commit** `feat(ui): add Streamlit shell with OIDC guard, container, caching, i18n, and idle timeout`.

### Task 8: M1 close-out

- [ ] **Step 1:** Update the docs from A6 that M1 touched: `doc/tech/02` §9 (auth seam) and `doc/tech/01` §2.1 (`kinbook-seed` defined).
- [ ] **Step 2:** `uv run pytest -q`, all linters, then `gh pr ready` and `gh pr checks --watch`. Expected: all green.
- [ ] **Step 3:** `gh pr merge --squash --delete-branch`; `git checkout main && git pull`.

## Deliverables after approval

1. Save this plan in the repo as `doc/plan/implementation-plan.md` (English). Commit on a `docs/implementation-plan` branch, open a PR, and squash-merge it (same workflow as PR #1).
2. Execute M1 (Part B) on `feat/m1-walking-skeleton`.
3. At the start of each later milestone, write `doc/plan/milestones/M<n>-<name>.md` with the same task format.

**Execution approach:** Subagent-driven development is recommended for M1. Every later milestone builds on these interfaces (engine, UoW, authz, container, query-budget harness), so a fresh per-task review catches foundation mistakes before they spread. M1 is 8 tasks.

## Verification

- Per task: the listed `uv run pytest …` commands pass. `ruff`, `mypy --strict`, `lint-imports`, and `semgrep` are clean.
- CI: `gh pr checks` reports `lint`, `unit`, `integration` (PostgreSQL 16), and `all-checks-passed` green.
- End-to-end (M1 exit): a local sign-in through the Keycloak dev realm reaches a home page showing the user's name. The DB contains the provisioned bundle. A second sign-in creates nothing new (verified by `test_second_sign_in_is_idempotent` and by row counts).
