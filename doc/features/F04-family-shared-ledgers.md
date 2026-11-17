# F04 — Family & Shared Ledgers

| Item | Value |
|------|-------|
| PRD | §6.4 (FM-1 … FM-10), §5 privacy rule |
| Priority | P0: FM-1 … FM-6, FM-10. P1: FM-7. P2: FM-8, FM-9 |
| Milestone | M3 |
| Pages | `family` (group, members, ledgers, balances), `join` (invitation landing) |
| Tech refs | [03 § 5, § 9](../tech/03-dataflow.md#5-settle-up-prd-flow-72-second-half), [04 § 3.1–3.3](../tech/04-database-schema.md#31-identity--family), [05 § 1.3, § 2.4](../tech/05-api-design.md#24-family--shared-ledgers-f04), [06 § 5.2](../tech/06-query-performance.md#52-family-balances-1-statement) |

## 1. Summary

A user creates a family group and invites members by link or QR code (shared over LINE). The group owns one or more **shared ledgers** (Household, Trip, Kids). Each shared expense records **who paid** and **how it is split**. Kinbook keeps a running balance of who owes whom and records settlements in one tap. Personal ledgers stay private.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-04.1 | household organizer | to invite my partner with a link I send on LINE | joining takes seconds |
| US-04.2 | member | a separate ledger for our Japan trip | trip costs don't mix with household costs |
| US-04.3 | payer | to record that I paid NT$2,480 for groceries split equally | my partner owes me half |
| US-04.4 | member | to see "Mei owes Jason NT$1,240" and settle with one tap | month-end is painless |
| US-04.5 | private person | my personal spending to stay invisible to the family | sharing doesn't cost privacy |
| US-04.6 | parent | to give my teenager read-only access | they learn about household costs |
| US-04.7 | member | to know when someone adds or changes a shared expense | nothing surprises me |

## 3. Requirement breakdown

| ID | Sub-feature | PRD | Priority | Acceptance criteria |
|----|-------------|-----|----------|---------------------|
| F04-1 | Create group | FM-1 | P0 | Creating a group creates a default shared ledger "Household" (currency = creator's default) with default categories, and adds the creator as `owner` and ledger member. A user can be in at most **one** group (v1). |
| F04-2 | Invite by link / QR | FM-1 | P0 | The owner generates an invitation (role `member` or `viewer`). It produces a URL with a 32-byte random token (only its SHA-256 is stored) and a QR image. It expires after 7 days and is single-use. The owner can revoke it. Sharing uses `st.code` (copy) + QR + a "Share via LINE" link (`https://line.me/R/share?text=…`). |
| F04-3 | Join flow | FM-1 | P0 | The invitee opens the link, signs in if needed, and sees the group name, inviter, role, and a **privacy explainer** ("Your personal ledger stays private; only entries you put in shared ledgers are visible"). Accept adds them to the group and to the default ledger. Expired, revoked, used, or over-limit invitations show a clear error. A user already in another group must leave it first. |
| F04-4 | Roles | FM-2 | P0 | `owner`: manage members, ledgers, invitations, transfer ownership. `member`: read/write shared ledgers they belong to. `viewer`: read-only (`ledger_members.can_write = false`). The last owner cannot leave without transferring ownership. |
| F04-5 | Multiple shared ledgers | FM-3 | P0 | The owner creates ledgers with name, currency, member subset, default split method, and default weights. Archive hides a ledger from quick add. The free plan allows 1 shared ledger and 2 members; Family Plus allows unlimited ledgers and 6 members (`MemberLimitReached`, `LedgerLimitReached`). |
| F04-6 | Ledger choice in quick add | FM-4 | P0 | The quick-add ledger selector defaults to the preference stored per category (`quick_add_preferences`), else Personal. It is updated on every save. |
| F04-7 | Payer & split | FM-5 | P0 | Split methods: **equal**, **percentage** (sum 100), **exact** (sum = amount), **shares** (positive weights). Members can be excluded from a split. Remainder allocation: largest remainder; ties go to the payer first, then by member join order. The UI shows each member's computed share live. |
| F04-8 | Balances & settle up | FM-6 | P0 | The Balances tab shows each member's net (positive = is owed) and minimal settlement suggestions. "Settle up" pre-fills debtor, creditor, and amount (editable; partial allowed). A settlement records `from_account` (debtor's) and optionally `to_account` (creditor's). Settlements are excluded from spending reports. |
| F04-9 | Activity & notifications | FM-7 | P1 (in-app P0) | Shared-ledger create, edit, delete, and settlement → `notifications` for the other ledger members (not the actor). Per-kind in-app/email settings in F08. The Activity tab lists `transaction_history` for the ledger with who/what/when. |
| F04-10 | Concurrent edits | FM-10 | P0 (Adapted) | Optimistic locking (`version`). On conflict, a dialog shows who changed it, with **Reload** and **Re-apply my change**. Field-level history is visible in the transaction detail. |
| F04-11 | Allowance for children | FM-8 | P2 | Scheduled allowance as a recurring transfer to the child's personal account. The child is a `viewer` in household ledgers. |
| F04-12 | Guest members | FM-9 | P2 | Trip ledgers may include guests (non-group users) with access limited to that ledger. Requires relaxing the one-group rule. |

## 4. Business rules

1. **Privacy:** Personal ledgers are visible only to their owner, with no exception, including the group owner. Shared ledgers are visible only to their `ledger_members`. Account names follow F03 rule 2.
2. Split shares must sum **exactly** to `ledger_amount_minor`, and every split user must be a ledger member.
3. Changing a ledger's member list or default split affects **future** transactions only.
4. Removing a member keeps their historical transactions and splits, so balances stay correct. Their net balance stays visible ("Former member" if they deleted their account) until settled.
5. A member with a non-zero balance can leave only after confirming a warning (`force=true`).
6. Balance arithmetic: `net(member) = Σ paid(expense, settlement) − Σ share(expense, settlement)`. The sum of all nets in a ledger is always **0**. This is an invariant test.
7. Shared-ledger income (e.g. a refund) is recorded without splits and does not affect balances (v1 simplification).

## 5. UI (Streamlit)

```
family  [Members] [Ledgers] [Balances] [Activity]
┌──────────────────────────────────────┐
│ Household ▾                          │
│ Mei      −1,240   owes               │
│ Jason    +1,240   is owed            │
│ ──────────────────────────────────── │
│ Suggested: Mei → Jason  NT$1,240     │
│              [ Settle up ]           │
└──────────────────────────────────────┘

add (shared ledger selected)
│ Paid by: (Mei | Jason)               │
│ Split:  [Equal|%|Exact|Shares]       │
│  ☑ Mei    1,240                      │
│  ☑ Jason  1,240                      │
```

| Widget key | Element |
|------------|---------|
| `family.create`, `family.name` | group creation |
| `family.invite.role`, `family.invite.create`, `family.invite.revoke.<id>` | invitation |
| `family.member.role.<user_id>`, `family.member.remove.<user_id>`, `family.transfer_owner` | member management |
| `family.ledger.create`, `family.ledger.edit.<id>`, `family.ledger.archive.<id>` | ledgers |
| `family.balances.ledger`, `family.settle.<from>.<to>` | balances |
| `join.accept`, `join.decline` | join page |
| `add.payer`, `add.split.method`, `add.split.include.<user_id>`, `add.split.value.<user_id>` | split editor in quick add |

## 6. Data & API

- Tables: `family_groups`, `family_members`, `invitations`, `ledgers`, `ledger_members`, `transactions`, `transaction_splits`, `transaction_history`, `notifications`, `outbox_events`, `quick_add_preferences`.
- Commands: `CreateFamilyGroup`, `RenameFamilyGroup`, `CreateInvitation`, `RevokeInvitation`, `AcceptInvitation`, `ChangeMemberRole`, `RemoveMember`, `LeaveGroup`, `TransferOwnership`, `CreateSharedLedger`, `UpdateSharedLedger`, `ArchiveLedger`, `RecordSettlement`.
- Queries: `GetFamilyOverview`, `PreviewInvitation`, `GetLedgerBalances`, `ListActivity`.
- Domain: `SplitAllocator`, `BalanceCalculator.suggest_settlements` (greedy max-creditor/max-debtor matching, ≤ n−1 transfers).

## 7. Performance

| Operation | Budget | Note |
|-----------|--------|------|
| `GetLedgerBalances` | 1 statement | CTE aggregate; settlement suggestions are computed in memory |
| `GetFamilyOverview` | 3 statements | members ⋈ users; ledgers with member counts (`GROUP BY`); open invitations |
| `ListActivity` | 1 statement | history ⋈ transactions ⋈ users, keyset paginated |
| Notification fan-out | 1 `INSERT … SELECT` from `ledger_members` per event batch | No loop over members |

## 8. Test checklist

- [ ] Hypothesis: for random sequences of expenses, splits, and settlements, Σ nets = 0, and applying all suggestions zeroes every net.
- [ ] Unit: each split method, exclusions, remainder to payer, exact-sum validation.
- [ ] Authz matrix: owner / member / viewer / non-member × read / write / manage on personal and shared ledgers.
- [ ] Integration: a group owner **cannot** read a member's personal ledger (returns `NotFound`).
- [ ] Integration: invitation token is single-use under concurrent accept (`SELECT … FOR UPDATE`).
- [ ] Playwright: two browser contexts (Mei, Jason). Jason adds a shared expense → Mei's bell shows 1 within 60 s → Mei settles → both balances are 0.
