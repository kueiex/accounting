# F07 — Data Management

| Item | Value |
|------|-------|
| PRD | §6.7 (DM-1 … DM-6) |
| Priority | P0: DM-2, DM-3, DM-5. P1: DM-4, DM-6. DM-1 dropped |
| Milestone | M5 (P0), M6 (P1) |
| Pages | `settings` › Data |
| Tech refs | [03 § 10](../tech/03-dataflow.md#10-export--account-deletion), [04 § 6](../tech/04-database-schema.md#6-migrations--data-lifecycle), [05 § 2.7](../tech/05-api-design.md#27-data-management-f07) |

## 1. Summary

Users own their data. They can export any ledger as CSV, Excel, or JSON, import history from other apps (P1), and delete their account self-service. Because Kinbook is a server-backed web app, data is always synced across devices. Offline mode is not available.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-07.1 | user | to download my transactions as Excel | I can do my own analysis or keep a backup |
| US-07.2 | switcher | to import my history from CWMoney or a spreadsheet | I don't start from zero |
| US-07.3 | user | to undo a bad import | mistakes are cheap |
| US-07.4 | user | to delete my account and data | I stay in control of my privacy |
| US-07.5 | user | to use the app on my phone and laptop | my data is the same everywhere |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Status | Priority | Acceptance criteria |
|----|-------------|-----|--------|----------|---------------------|
| F07-1 | Offline-first | DM-1 | **Dropped** | — | Not possible with Streamlit. Mitigation: quick-add input is kept in session state across reruns. If the connection drops briefly, Streamlit reconnects to the same server session and the unsent form values survive. If the server process restarted, the draft is lost, and the user sees a "Connection lost — please re-enter" notice. |
| F07-2 | Cross-device sync | DM-2 | Keep (inherent) | P0 | Any change is visible on another device of the same user on its next rerun (or ≤ 60 s via the polling fragment). |
| F07-3 | Export | DM-3 | Keep | P0 | Choose ledger, date range (default: all), and format (CSV UTF-8 with BOM for Excel, XLSX, JSON full fidelity). Columns: date, type, amount, currency, ledger amount, category, subcategory, account (own accounts only), payer, my share, note, merchant, tags. Files ≤ 20 MB are delivered via `st.download_button`; larger files via a pre-signed URL valid for 15 min. Every export is written to the security log. |
| F07-4 | CSV import | DM-4 | Keep | P1 | Upload CSV/XLSX → choose a preset (Generic, CWMoney, Moneybook) or map columns manually → preview the first 50 parsed rows with errors → map unknown categories to existing ones or create new → commit. Max 50,000 rows per file. Duplicates (same date, amount, note) are flagged, not silently dropped. |
| F07-5 | Undo import | DM-4 | Keep | P1 | Each commit creates an `import_batch`. "Undo" soft-deletes all its transactions in one statement. Available for 30 days. |
| F07-6 | Account deletion | DM-5 | Keep | P0 | Settings › Delete account → explains consequences (personal data removed; shared-ledger entries stay for other members, attributed to "Former member") → type the confirmation phrase → signed out. The user can cancel by signing in within 30 days. A group owner must transfer ownership or delete the group first. |
| F07-7 | Local backup | DM-6 | Adapted | P1 | Replaced by the JSON full export (all ledgers the user owns, accounts, categories, budgets, rules). A JSON re-import into a new account is P2. |

## 4. Business rules

1. Exports include only data the actor can read. Other members' private account names are never exported.
2. Import writes to one ledger chosen by the user. It never writes into shared ledgers without explicit selection.
3. Imported rows get `source = 'import'` and `import_batch_id`. They go through the same domain validation as manual entries.
4. Account deletion is irreversible after 30 days. Backups older than 30 days roll off naturally (PITR 7 days, dumps 30 days).

## 5. UI (Streamlit)

| Widget key | Element |
|------------|---------|
| `data.export.ledger`, `data.export.range`, `data.export.format`, `data.export.go`, `data.export.download` | export |
| `data.import.file`, `data.import.preset`, `data.import.map.<column>`, `data.import.preview`, `data.import.commit` | import wizard (`st.file_uploader`, `st.data_editor` for mapping) |
| `data.import.history`, `data.import.undo.<batch_id>` | import history |
| `data.delete.start`, `data.delete.phrase`, `data.delete.confirm` | deletion |

## 6. Data & API

- Tables: `transactions`, `import_batches`, `users.deletion_requested_at`, all user-owned tables for purge.
- Commands: `ExportLedger`, `CommitCsvImport`, `UndoImport`, `RequestAccountDeletion`, `CancelAccountDeletion`, `PurgeDeletedUsers` (worker), `ExpireExports` (worker).
- Queries: `PreviewCsvImport`.

## 7. Performance

- Export: **one** joined statement streamed with `yield_per(2000)`. Names (category, parent category, account, payer, tags via `string_agg`) are joined in, so there are no per-row lookups. Memory stays bounded.
- Import commit: parse and validate in memory (pandas). Resolve all categories and accounts with **one** query each. Insert transactions in batches of 1,000 with multi-row `INSERT`, then bump the ledger revision once. A 50 k-row import takes ≤ 30 s and runs in the worker if it has more than 5,000 rows (the UI polls the batch status).
- Purge: `DELETE … WHERE id = ANY(:batch)` in chunks of 1,000 to keep locks short.

## 8. Test checklist

- [ ] Integration: exported CSV re-parses to identical totals (round-trip).
- [ ] Integration: export of a shared ledger hides other members' private account names.
- [ ] Integration: import query count is independent of row count (except insert batches).
- [ ] Integration: undo import removes exactly the batch rows.
- [ ] Integration: after purge, no rows reference the user except anonymized shared-ledger history.
