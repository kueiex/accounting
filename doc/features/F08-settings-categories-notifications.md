# F08 — Settings, Categories & Notifications

| Item | Value |
|------|-------|
| PRD | §6.8 (ST-1 … ST-4), FM-7 (notification delivery) |
| Priority | P0: ST-1, ST-2, ST-4, notification center. P1: ST-3, email notifications |
| Milestone | M2 (categories, settings), M3 (notification center) |
| Pages | `settings` (Profile, Categories, Notifications, Data, E-invoice, About), header notification bell |
| Tech refs | [02 § 6](../tech/02-system-design.md#6-notifications), [04 § 3.2, § 3.5](../tech/04-database-schema.md#35-e-invoice-fx-notifications-outbox-preferences), [05 § 2.8](../tech/05-api-design.md#28-settings-categories-notifications-f08) |

## 1. Summary

Settings let users personalize Kinbook: categories per ledger (add, rename, reorder, icon, color, merge, archive), theme and text size, week and month start, and a daily logging reminder. A notification center collects shared-ledger activity, budget alerts, reminders, and summaries, with per-kind in-app and email controls.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-08.1 | user | categories that match how my family talks about money (e.g. "Red envelopes", "Pet") | logging feels natural |
| US-08.2 | user | to merge two similar categories | reports stay clean |
| US-08.3 | user paid on the 5th | months to start on the 5th | budgets match my pay cycle |
| US-08.4 | forgetful user | a reminder at 21:30 if I haven't logged today | I keep the habit |
| US-08.5 | member | to see what changed in our shared ledger since I last looked | I stay in sync |
| US-08.6 | user | to choose which notifications reach my email | I'm not spammed |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Priority | Acceptance criteria |
|----|-------------|-----|----------|---------------------|
| F08-1 | Category management | ST-1 | P0 | Per ledger and kind: add (name ≤ 40, icon emoji, color), rename, change parent (depth ≤ 2), reorder (▲▼), archive/unarchive. A name must be unique within its parent. Shared-ledger categories are editable by members with `can_write`. |
| F08-2 | Merge categories | ST-1 | P0 | Merge A → B moves all transactions and budgets of A to B (one `UPDATE` each) and archives A. If B already has a budget for the same period, A's budget is dropped (the dialog says so). Only same-kind categories in the same ledger can be merged. A confirmation shows the count of affected transactions. |
| F08-3 | Theme & text size | ST-2 | P0 (Adapted) | Theme `system`/`light`/`dark` is stored in `users.theme` and applied with a Streamlit theme config. Text size follows browser zoom and OS settings; layout is tested at 200 % zoom. |
| F08-4 | Locale, time zone, currency | — | P0 | Changeable language (zh-TW/en), time zone, and default currency (for new personal entries and net worth). Changing the default currency does **not** convert existing data. |
| F08-5 | Week & month start | ST-3 | P1 | Week start (Mon/Sun) and month start day (1–28) affect reports and budgets for personal ledgers. Shared ledgers use the group owner's setting. |
| F08-6 | Daily logging reminder | ST-4 | P0 | Optional time (15-min steps). If the user has logged nothing today (local date) by that time, they get a `logging_reminder` notification (+ email if enabled). Sent at most once per day. |
| F08-7 | Notification center | FM-7 | P0 | A bell in the header shows the unread count (`@st.fragment(run_every=60)`). Opening it lists the newest 30 notifications with deep links (a transaction opens its detail dialog). "Mark all read". Kinds: `shared_tx_added`, `shared_tx_changed`, `settlement`, `invite_accepted`, `budget_threshold`, `recurring_due`, `card_due`, `einvoice_new`, `monthly_summary`, `logging_reminder`, `goal_achieved`. |
| F08-8 | Notification preferences | FM-7 | P0 in-app / P1 email | Per kind: in-app on/off and email on/off. Defaults: all in-app on; email on only for `budget_threshold` and `monthly_summary`. |

## 4. Business rules

1. Default categories are seeded per ledger with stable `slug`s. Renaming keeps the slug, so preferences and moves between ledgers still map.
2. An archived category disappears from pickers but stays on historical transactions and in reports.
3. A category with active children cannot be archived until its children are archived or moved.
4. Notifications are created only by the worker from outbox events (or scheduled jobs). The UI never creates notifications directly.
5. Read notifications are deleted after 180 days.

## 5. UI (Streamlit)

```
settings  [Profile] [Categories] [Notifications] [Data] [E-invoice] [About]

Categories — Household · Expense
┌──────────────────────────────────────┐
│ 🍽 Food                    ▲ ▼ ✎     │
│    ├ Breakfast             ▲ ▼ ✎     │
│    ├ Lunch                 ▲ ▼ ✎     │
│    └ Groceries             ▲ ▼ ✎     │
│ 🚇 Transport               ▲ ▼ ✎     │
│ [+ Add] [Merge…] [Show archived]     │
└──────────────────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `settings.tabs` | `st.tabs` |
| `settings.profile.*` | name, locale, time zone, currency, theme, week start, month start, reminder time |
| `settings.categories.ledger`, `settings.categories.kind` | scope |
| `settings.categories.add`, `settings.categories.edit.<id>`, `settings.categories.up.<id>`, `settings.categories.down.<id>`, `settings.categories.merge` | actions |
| `settings.notifications.<kind>.in_app`, `settings.notifications.<kind>.email` | toggles (`st.data_editor` table of kinds) |
| `header.bell`, `header.bell.mark_all` | notification popover (`st.popover`) |
| `settings.signout` | sign out |

## 6. Data & API

- Tables: `categories`, `users`, `user_notification_settings`, `notifications`, `outbox_events`.
- Commands: `CreateCategory`, `UpdateCategory`, `ReorderCategories`, `MergeCategories`, `ArchiveCategory`, `UpdateProfile`, `UpdateNotificationSettings`, `MarkNotificationsRead`, `SendLoggingReminders` (worker), `DispatchOutbox` (worker).
- Queries: `ListCategories`, `ListNotifications`, `CountUnreadNotifications`.

## 7. Performance

- `ListCategories`: 1 statement, tree assembled in memory (≤ 200 categories per ledger).
- Bell: `SELECT count(*) FROM notifications WHERE user_id = :u AND read_at IS NULL` via the partial index `notifications_unread_idx`. It costs ~1 ms per minute per open session.
- `SendLoggingReminders`: one statement selects users whose `reminder_time` falls in the current 5-min window **and** who have no transaction created today (`NOT EXISTS` on `transactions_creator_recent_idx`). Notifications are inserted with one `INSERT … SELECT`.
- Reorder: one `UPDATE … FROM (VALUES …)`.

## 8. Test checklist

- [ ] Unit: depth ≤ 2, unique name within parent, same-kind merge.
- [ ] Integration: merge moves transactions and budgets in ≤ 3 statements.
- [ ] Integration: logging reminder skips users who logged today and is sent at most once.
- [ ] AppTest: bell shows the unread count and "mark all read" clears it.
- [ ] Accessibility: settings pages usable at 200 % zoom on a 375 px viewport.
