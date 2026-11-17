# 06 — Query Performance & N+1 Prevention

## 1. Why this matters in a Streamlit app

Streamlit reruns the whole script on every interaction. Suppose a page renders a list like this:

```python
# ANTI-PATTERN: N+1 inside a Streamlit loop
for tx in transactions:                       # 1 query
    st.write(tx.category.name)                # +1 lazy load per row
    st.write(tx.account.name)                 # +1 per row
    st.write(", ".join(t.name for t in tx.tags))  # +1 per row
```

It issues 1 + 3N queries **on every click**, for every family member's session. With 50 rows that is 151 round trips per rerun. Kinbook makes this pattern impossible by construction, then measures it in CI.

## 2. Rules

| # | Rule |
|---|------|
| R1 | **No lazy loading.** Every ORM `relationship()` is declared with `lazy="raise"`. Touching an unloaded relationship raises `InvalidRequestError` in dev, tests, and prod. |
| R2 | **Repositories declare load plans.** Each aggregate repository method has a fixed eager-load plan: `selectinload` for collections, `joinedload` for many-to-one. Callers cannot add or forget loads. |
| R3 | **Read screens use query services, not aggregates.** Query services issue explicit SQLAlchemy Core `select()` statements with joins and aggregates and return flat Pydantic DTOs. DTOs have no relationships to traverse. |
| R4 | **Constant query count per screen.** A screen section costs O(1) queries regardless of row count. Child data for a page of rows is fetched with **one** batch query (`WHERE parent_id = ANY(:ids)`), never one per row. |
| R5 | **Aggregate in SQL, not Python.** Sums, counts, and group-bys run in PostgreSQL (`SUM`, `GROUP BY`, `FILTER`). Python never loads rows only to add them up. |
| R6 | **No `get()` in a loop.** Repositories expose `get_many(ids)`. A review checklist item and a Semgrep rule run in pre-commit and CI flag repository `.get(` calls inside `for` loops and comprehensions. Ruff does not support custom rules, so Semgrep covers this. |
| R7 | **Keyset pagination**, never `OFFSET` on large tables: `WHERE (occurred_on, id) < (:last_date, :last_id) ORDER BY occurred_on DESC, id DESC LIMIT 50`. |
| R8 | **Every hot `WHERE` / `ORDER BY` has an index.** These are partial (`deleted_at IS NULL`) and covering (`INCLUDE`) where it avoids heap reads. See [04 § Transaction indexes](04-database-schema.md#transaction-indexes-hot-paths). |
| R9 | **Bulk writes.** Child rows (splits, tags) are inserted with a single multi-row `INSERT`. Imports use batches of 1,000 rows. |
| R10 | **Cache reads keyed by ledger revision.** Unchanged ledgers cost one primary-key lookup per rerun (see [02 § Caching](02-system-design.md#4-caching-strategy)). |
| R11 | **Measure.** Every query service and use case has a **query budget** asserted in integration tests. In dev, a warning is logged when a single Streamlit rerun exceeds 15 statements. |

## 3. ORM configuration

```python
# infrastructure/db/orm_models.py (excerpt)
class TransactionRow(Base):
    __tablename__ = "transactions"
    id: Mapped[UUID] = mapped_column(primary_key=True)
    ledger_id: Mapped[UUID] = mapped_column(ForeignKey("ledgers.id"))
    # ...
    splits: Mapped[list["SplitRow"]] = relationship(lazy="raise", cascade="all, delete-orphan")
    tags: Mapped[list["TagRow"]] = relationship(secondary="transaction_tags", lazy="raise")
    attachments: Mapped[list["AttachmentRow"]] = relationship(lazy="raise")
    category: Mapped["CategoryRow | None"] = relationship(lazy="raise")
```

A unit test walks `Base.registry.mappers` and fails if any relationship's `lazy` strategy is not `"raise"`:

```python
def test_all_relationships_raise_on_lazy_load() -> None:
    offenders = [
        f"{m.class_.__name__}.{r.key}"
        for m in Base.registry.mappers
        for r in m.relationships
        if r.lazy != "raise"
    ]
    assert offenders == []
```

## 4. Write side: repository load plans

```python
# infrastructure/db/repositories/transactions.py
_TX_AGGREGATE_LOADS = (
    selectinload(TransactionRow.splits),
    selectinload(TransactionRow.tags),
    selectinload(TransactionRow.attachments),
)

class SqlTransactionRepository(TransactionRepository):
    def get(self, tx_id: TransactionId) -> Transaction:
        row = self._s.scalars(
            select(TransactionRow).where(TransactionRow.id == tx_id).options(*_TX_AGGREGATE_LOADS)
        ).one_or_none()                                     # 1 + 3 = 4 statements, constant
        if row is None:
            raise NotFound(tx_id)
        return to_domain(row)

    def get_many(self, ids: Sequence[TransactionId]) -> list[Transaction]:
        rows = self._s.scalars(
            select(TransactionRow).where(TransactionRow.id.in_(ids)).options(*_TX_AGGREGATE_LOADS)
        ).all()                                             # still 4 statements for any len(ids)
        return [to_domain(r) for r in rows]
```

`selectinload` issues one `IN (...)` query per collection for the whole result set. This is the ORM-level batching that replaces per-row lazy loads.

## 5. Read side: query service patterns

### 5.1 Transaction list page (2 statements for any page size)

```python
# infrastructure/db/query_services/transactions.py
def list_page(self, actor: UserId, f: TxFilter, cursor: Cursor | None) -> TxPage:
    t, c, pc, a, u = transactions, categories, categories.alias("pc"), accounts, users
    stmt = (
        select(
            t.c.id, t.c.type, t.c.occurred_on, t.c.amount_minor, t.c.currency,
            t.c.ledger_amount_minor, t.c.note, t.c.merchant, t.c.version,
            c.c.name.label("category_name"), c.c.icon, pc.c.name.label("parent_category_name"),
            a.c.name.label("account_name"), u.c.display_name.label("payer_name"),
        )
        .join(c, c.c.id == t.c.category_id, isouter=True)
        .join(pc, pc.c.id == c.c.parent_id, isouter=True)
        .join(a, a.c.id == t.c.account_id, isouter=True)
        .join(u, u.c.id == t.c.payer_user_id)
        .where(t.c.ledger_id == f.ledger_id, t.c.deleted_at.is_(None), *f.to_sql(t))
        .order_by(t.c.occurred_on.desc(), t.c.id.desc())
        .limit(PAGE_SIZE + 1)
    )
    if cursor:
        stmt = stmt.where(tuple_(t.c.occurred_on, t.c.id) < (cursor.occurred_on, cursor.id))
    rows = self._conn.execute(stmt).all()                                   # statement 1

    ids = [r.id for r in rows[:PAGE_SIZE]]
    extras = self._conn.execute(                                             # statement 2
        select(
            transactions.c.id,
            select(func.array_agg(tags.c.name)).join(transaction_tags)
                .where(transaction_tags.c.transaction_id == transactions.c.id)
                .scalar_subquery().label("tag_names"),
            select(func.count()).select_from(transaction_splits)
                .where(transaction_splits.c.transaction_id == transactions.c.id)
                .scalar_subquery().label("split_count"),
            select(func.count()).select_from(attachments)
                .where(attachments.c.transaction_id == transactions.c.id)
                .scalar_subquery().label("attachment_count"),
        ).where(transactions.c.id == any_(ids))
    ).all()
    return assemble_page(rows, extras, PAGE_SIZE)
```

Authorization (`actor` may read `f.ledger_id`) is checked by the caller before this runs, and the `ledger_id` predicate guarantees no cross-ledger leakage. In shared ledgers, `account_name` is selected as `CASE WHEN a.owner_user_id = :actor OR EXISTS (account_shares …) THEN a.name END`. Another member's private account name is therefore never sent to the UI (F03 rule 2). This is still the same single statement.

### 5.2 Family balances (1 statement)

```sql
WITH scope AS (
    SELECT id, payer_user_id, ledger_amount_minor
    FROM transactions
    WHERE ledger_id = :ledger_id AND deleted_at IS NULL AND type IN ('expense','settlement')
),
paid AS (
    SELECT payer_user_id AS user_id, SUM(ledger_amount_minor) AS amt FROM scope GROUP BY payer_user_id
),
owed AS (
    SELECT s.user_id, SUM(s.share_minor) AS amt
    FROM transaction_splits s JOIN scope ON scope.id = s.transaction_id
    GROUP BY s.user_id
)
SELECT m.user_id, u.display_name,
       COALESCE(p.amt, 0) - COALESCE(o.amt, 0) AS net_minor
FROM ledger_members m
JOIN users u ON u.id = m.user_id
LEFT JOIN paid p ON p.user_id = m.user_id
LEFT JOIN owed o ON o.user_id = m.user_id
WHERE m.ledger_id = :ledger_id;
```

The domain `BalanceCalculator.suggest_settlements(nets)` then computes the minimal "who pays whom" list in memory. This needs no further queries.

### 5.3 Account balances for all of a user's accounts (1 statement)

```sql
SELECT a.id, a.name, a.type, a.currency,
       a.opening_balance_minor
     + COALESCE(o.delta, 0)
     + COALESCE(i.delta, 0) AS balance_minor
FROM accounts a
LEFT JOIN LATERAL (
    SELECT SUM(CASE
                 WHEN t.type = 'income'                      THEN  t.account_amount_minor
                 WHEN t.type = 'adjustment'                  THEN  t.adjustment_sign * t.account_amount_minor
                 ELSE                                              -t.account_amount_minor
               END) AS delta
    FROM transactions t
    WHERE t.account_id = a.id AND t.deleted_at IS NULL
) o ON true
LEFT JOIN LATERAL (
    SELECT SUM(t.to_account_amount_minor) AS delta
    FROM transactions t
    WHERE t.to_account_id = a.id AND t.deleted_at IS NULL
) i ON true
WHERE a.owner_user_id = :actor_id AND a.archived_at IS NULL
ORDER BY a.sort_order;
```

Both lateral sub-selects are answered from the covering indexes `transactions_account_idx` and `transactions_to_account_idx` with index-only scans.

### 5.4 Category breakdown with parent roll-up (1 statement)

```sql
SELECT COALESCE(c.parent_id, c.id)            AS top_category_id,
       c.id                                   AS category_id,
       SUM(t.ledger_amount_minor)             AS total_minor,
       COUNT(*)                               AS tx_count
FROM transactions t
JOIN categories c ON c.id = t.category_id
WHERE t.ledger_id = :ledger_id
  AND t.deleted_at IS NULL
  AND t.type = :kind                          -- 'expense' or 'income'
  AND t.occurred_on >= :from AND t.occurred_on < :to
GROUP BY GROUPING SETS ((COALESCE(c.parent_id, c.id)), (COALESCE(c.parent_id, c.id), c.id));
```

### 5.5 Dashboard (home) — total 4 statements

| # | Statement | Index used |
|---|-----------|-----------|
| 1 | `SELECT revision FROM ledgers WHERE id = :id` (cache key) | PK |
| 2 | Month totals: `SUM(...) FILTER (WHERE type='expense')`, `FILTER (WHERE type='income')` for this month and last month in one pass | `transactions_ledger_date_idx` |
| 3 | Budget progress for all budgets of the ledger (join budgets ← aggregated spend per category including children) | `budgets_ledger_idx`, `transactions_ledger_cat_date_idx` |
| 4 | Recent 20 transactions (pattern 5.1, statement 1 only; tags are not shown on home) | `transactions_ledger_date_idx` |

On a cache hit only statement 1 runs.

## 6. Enforcement

### 6.1 Query counter fixture

```python
# tests/integration/conftest.py
@contextmanager
def count_queries(engine: Engine) -> Iterator[list[str]]:
    statements: list[str] = []
    def _on_execute(conn, cursor, statement, params, context, executemany):  # noqa: ANN001
        statements.append(statement)
    event.listen(engine, "before_cursor_execute", _on_execute)
    try:
        yield statements
    finally:
        event.remove(engine, "before_cursor_execute", _on_execute)

@pytest.fixture
def assert_max_queries(engine: Engine):
    @contextmanager
    def _assert(n: int) -> Iterator[None]:
        with count_queries(engine) as stmts:
            yield
        assert len(stmts) <= n, f"expected <= {n} queries, got {len(stmts)}:\n" + "\n---\n".join(stmts)
    return _assert
```

### 6.2 Budget tests are parametrized by data size

The test seeds **1 row and 200 rows** and asserts the *same* budget. This catches N+1 even when the budget is generous.

```python
@pytest.mark.parametrize("n_rows", [1, 200])
def test_transaction_list_query_budget(n_rows, seed_ledger, tx_queries, assert_max_queries):
    ledger = seed_ledger(transactions=n_rows, tags_per_tx=2, splits=True)
    with assert_max_queries(2):
        page = tx_queries.list_page(ledger.owner, TxFilter(ledger_id=ledger.id), cursor=None)
    assert len(page.items) == min(n_rows, 50)
```

### 6.3 Query budgets (contract)

| Use case / query | Max statements |
|------------------|----------------|
| `DashboardQueries.get` | 3 (+1 revision lookup) |
| `TransactionQueries.list_page` | 2 |
| `TransactionQueries.detail` (incl. history, attachments) | 3 |
| `QuickAddQueries.context` (accounts, recent categories, ledgers, preferences) | 4 |
| `ReportQueries.monthly_overview` / `category_breakdown` / `trend` | 1 each |
| `FamilyQueries.balances` | 1 |
| `AccountQueries.list_with_balances` | 1 |
| `BudgetQueries.progress` | 1 |
| `RecordTransaction` (shared, with splits and tags) | ≤ 9: authz, ledger lock+revision bump, account/category validation, insert tx, insert splits, insert tags, history, outbox, quick-add pref upsert |
| `RunDueRecurringRules` per batch of 500 rules | ≤ 8 (independent of batch size) |

Raising a budget requires a code-review approval that explains why.

### 6.4 Runtime guard

In `local` and `staging`, a per-rerun counter (a `contextvars` value set by the page bootstrap and incremented in the `before_cursor_execute` listener) logs a `WARNING n_plus_one_suspect` with the page name when one rerun exceeds 15 statements. In `prod` the count is emitted as a metric (`db_queries_per_rerun`) instead.

## 7. Batch jobs

Worker jobs are written set-based:

```python
# application/use_cases/transactions/run_due_recurring_rules.py (shape)
def __call__(self, today: date) -> int:
    with self._uow:
        rules = self._uow.recurring.lock_due(today, limit=500)       # SELECT ... FOR UPDATE SKIP LOCKED
        ledgers = self._uow.ledgers.get_many({r.ledger_id for r in rules})                 # 1 query
        members = self._uow.ledgers.members_for({r.ledger_id for r in rules})              # 1 query
        occurrences = [occ for r in rules for occ in r.due_occurrences(today)]             # pure domain
        txs = [r.materialize(occ, ledgers[r.ledger_id], members[r.ledger_id]) for r, occ in occurrences]
        self._uow.transactions.add_many(txs)                         # multi-row INSERTs
        self._uow.recurring.record_occurrences(occurrences)          # ON CONFLICT DO NOTHING
        self._uow.ledgers.bump_revisions({t.ledger_id for t in txs}) # 1 UPDATE ... WHERE id = ANY
        self._uow.outbox.add_many(events_for(txs))
        self._uow.commit()
    return len(txs)
```

## 8. Scaling triggers

Optimizations are added only when a metric crosses a threshold:

| Signal | Threshold | Action |
|--------|-----------|--------|
| p95 of any report query | > 150 ms for 3 days | Add `ledger_monthly_category_totals` summary table, maintained incrementally in the same transaction as writes |
| Largest ledger | > 200 k live transactions | Same as above, plus monthly partitioning of `transactions` by `occurred_on` |
| DB CPU | > 60 % sustained | Read replica for query services (query services already take a separate read engine) |
| Cache hit ratio (`st.cache_data`) | < 60 % | Review cache keys; raise TTL |

## 9. Connection & statement settings

| Setting | App | Worker |
|---------|-----|--------|
| SQLAlchemy `pool_size` / `max_overflow` | 5 / 5 per replica | 3 / 2 |
| `pool_pre_ping` | on | on |
| `statement_timeout` | 5 s | 60 s |
| `idle_in_transaction_session_timeout` | 10 s | 60 s |
| `lock_timeout` | 2 s | 5 s |
| Isolation level | READ COMMITTED (ledger row lock serializes ledger writes) | READ COMMITTED |

Sessions are short: one Unit of Work per use case call. A DB connection is never held across a Streamlit rerun or while waiting for user input.
