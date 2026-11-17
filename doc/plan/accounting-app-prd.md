# Product Requirements Document — Personal & Family Accounting App

| Item | Value |
|------|-------|
| Working name | **Kinbook** (placeholder) |
| Document owner | Product Management |
| Status | Draft v0.1 |
| Last updated | 2026-11-17 |
| Target market | Taiwan first (zh-TW, TWD), English UI from day one |
| Platforms | Android & iOS (mobile-first), read-only web dashboard later |

> Assumptions made by PM are marked **[PM call]**. They are proposals, not validated decisions; revisit them after user interviews (see §13).

---

## 1. Background & Problem

Most households track money in one of three ways: a spreadsheet one person maintains, a single-user expense app, or not at all. Each breaks down for families:

1. **Recording is too slow.** If logging a purchase takes more than a few seconds, people stop after a week. The data becomes incomplete and useless.
2. **Single-user apps don't fit households.** Partners and family members spend from different cards and wallets. One person ends up re-typing everyone's receipts, or the family has no combined picture.
3. **"Who paid for what" causes friction.** Shared costs (groceries, rent, utilities, trips) are paid by whoever is at the counter. Settling up at month-end is manual and error-prone.
4. **Privacy is all-or-nothing.** Family members want a shared household view without exposing every personal purchase.
5. **Local context is missing.** In Taiwan, many purchases already produce a cloud e-invoice (雲端發票) on a mobile barcode carrier (手機條碼載具), yet most apps ignore this ready-made data source.

## 2. Vision

> *The fastest way for a household to know where its money goes — together, without giving up personal privacy.*

Kinbook is a mobile app where each person keeps their own books and the family shares a household ledger. Logging a purchase takes under 5 seconds; month-end shows one clear picture of household spending and who owes whom.

## 3. Goals, Non-Goals, Anti-Goals

### 3.1 Goals

| # | Goal | Measured by |
|---|------|-------------|
| G1 | Make recording effortless | Median time to log a manual expense ≤ 5 s |
| G2 | Build a lasting habit | D30 retention ≥ 35 %; ≥ 4 logging days / week among active users |
| G3 | Give households one shared picture | ≥ 40 % of active users belong to a family group with ≥ 2 members |
| G4 | Remove settle-up friction | ≥ 70 % of shared expenses settled inside the app within 30 days |
| G5 | Earn trust with financial data | Zero data-breach incidents; App Store / Play rating ≥ 4.5 |

### 3.2 Non-Goals (out of scope for v1)

- Business / SME bookkeeping, double-entry accounting, invoicing customers, payroll.
- Tax filing or tax-return preparation.
- Investment portfolio tracking (stock prices, P&L, dividends). Users may hold an "Investment" account with a manually updated balance only.
- Direct bank-account aggregation / open-banking connections (regulatory and partner cost too high for v1). **[PM call]**
- Payments or money transfer. The app *records* settlements; it does not move money.
- Desktop-first experience.

### 3.3 Anti-Goals (would be a failure even if it "works")

- Selling or sharing user financial data with advertisers or third parties.
- A cluttered, accountant-style UI that intimidates non-experts.
- Family features that let one member see another's private transactions without explicit consent.
- Forcing an account sign-up before a person can log their first expense.
- Losing data when the device is offline or the user switches phones.

## 4. Target Users & Personas

| Persona | Description | Key needs |
|---------|-------------|-----------|
| **Mei — the household organizer** (34, married, one child) | Manages the family budget, currently in a Google Sheet she updates on Sunday nights. | Combined household view, budgets per category, monthly report, less manual typing. |
| **Jason — the reluctant partner** (36) | Pays for many shared things on his credit card but never logs them. | Logging must be near-zero effort; e-invoice auto-import; doesn't want his hobby spending exposed. |
| **Amy — the young professional** (25, lives alone or with roommates) | Wants to save for travel; splits rent and utilities with roommates. | Personal budget, savings goal, split bills with non-family housemates. |
| **Grandpa Lin — the occasional user** (68) | Gives children allowance, pays some household bills in cash. | Large text, simple flow, minimal settings. |

**Primary persona for v1:** Mei + Jason (a couple sharing a household). **[PM call]**

## 5. Key Concepts (Domain Model)

| Concept | Definition |
|---------|------------|
| **User** | A person with a Kinbook identity. Can use the app without sign-up (local mode) until they want sync or family features. |
| **Ledger** | A book of transactions. Every user has one **Personal ledger**. A **Family group** owns one or more **Shared ledgers** (e.g. "Household", "Japan Trip 2027"). |
| **Family group** | A set of members who share ledgers. Roles: **Owner**, **Member**, **Viewer**. |
| **Account** | Where money lives: cash, bank account, credit card, e-wallet (LINE Pay, JKOPAY, Easy Card, iPASS), investment (balance-only), loan. Accounts belong to a single user. |
| **Transaction** | Expense, income, or transfer. Has amount, currency, date, account, category, note, tags, payer, optional attachments, optional split. |
| **Category** | Two-level hierarchy (e.g. Food › Groceries). Default set provided; fully customizable per ledger. |
| **Split** | Allocation of one shared expense across members (equal, by percentage, by exact amount, or by shares). |
| **Balance / Settlement** | Net amount each member owes or is owed within a shared ledger; a settlement transaction zeroes it out. |
| **Budget** | Spending limit per period (monthly default) for a ledger, optionally per category. |
| **Goal** | A savings target with amount and deadline, tracked against a chosen account. |

### Privacy rule (core principle)

- A transaction in a **Personal ledger** is visible only to its owner. Always.
- A transaction in a **Shared ledger** is visible to all members of that ledger.
- A user may **move** a personal transaction into a shared ledger (or log directly into it). They may never be forced to.
- Account balances are private by default; a user may opt to share specific accounts (e.g. a joint bank account) with the family.

## 6. Feature Requirements

Priority: **P0** = MVP must-have, **P1** = v1.x fast follow, **P2** = later.

### 6.1 Onboarding & Identity

| ID | Requirement | Priority |
|----|-------------|----------|
| ON-1 | User can log the first expense within 30 s of first launch, with no sign-up (local mode). | P0 |
| ON-2 | Choose language (zh-TW / English) and default currency (TWD default) on first launch. | P0 |
| ON-3 | Optional account creation via Apple, Google, LINE login, or email + OTP. Local data migrates to the cloud on sign-up. | P0 (Apple/Google/email), P1 (LINE) |
| ON-4 | Guided setup: add 1–3 accounts, pick a monthly budget (skippable). | P0 |
| ON-5 | App lock: PIN and biometric (fingerprint / Face ID). | P0 |

### 6.2 Recording Transactions

| ID | Requirement | Priority |
|----|-------------|----------|
| TX-1 | Quick-add screen: numeric keypad opens immediately; amount → category (recent/favorite first) → save. Date defaults to now, account defaults to last used. | P0 |
| TX-2 | Expense, income, and transfer between own accounts. | P0 |
| TX-3 | Optional fields: note, tags, location, photo of receipt (≤ 5 attachments). | P0 (note, tags, photo), P1 (location) |
| TX-4 | Edit, duplicate, and delete transactions; deleted items go to a 30-day trash. | P0 |
| TX-5 | Recurring transactions (daily / weekly / monthly / yearly / custom) with reminder or auto-post. | P0 |
| TX-6 | Multi-currency: record in any currency; converted to ledger currency at the transaction-date rate; user may override the rate. | P1 |
| TX-7 | Home-screen widget and quick-settings tile for one-tap add. | P1 |
| TX-8 | **Taiwan e-invoice import**: link the mobile barcode carrier; invoices import as draft transactions with merchant and line items; user confirms category in a review inbox. | P1 |
| TX-9 | Receipt OCR: take a photo → amount, date, merchant prefilled. | P2 |
| TX-10 | Voice entry ("午餐 150 元"), natural-language parsing to a draft transaction. | P2 |
| TX-11 | Smart category suggestion based on merchant/note history. | P1 |
| TX-12 | Search and filter by text, date range, amount range, category, account, tag, member. | P0 |

### 6.3 Accounts

| ID | Requirement | Priority |
|----|-------------|----------|
| AC-1 | Create, edit, archive accounts by type (cash, bank, credit card, e-wallet, investment, loan). | P0 |
| AC-2 | Running balance per account; manual balance adjustment creates an "adjustment" transaction. | P0 |
| AC-3 | Credit card: statement day and due day, with a payment-due reminder. | P1 |
| AC-4 | Net-worth view (assets − liabilities) for the user's own accounts. | P1 |
| AC-5 | Share a specific account (e.g. joint account) with the family group, read-only or read-write. | P1 |

### 6.4 Family & Shared Ledgers

| ID | Requirement | Priority |
|----|-------------|----------|
| FM-1 | Create a family group; invite via link, QR code, or LINE share. Invitee must have an account. | P0 |
| FM-2 | Roles: Owner (manage members, delete ledger), Member (add/edit own entries), Viewer (read-only, e.g. a child). | P0 |
| FM-3 | Create multiple shared ledgers within a group (Household, Trip, Kids). | P0 |
| FM-4 | When adding a transaction, choose ledger: Personal (default) or a shared ledger. Last choice is remembered per category. | P0 |
| FM-5 | Each shared transaction records **who paid** and **how it is split** (default: equal among ledger members, configurable per ledger). | P0 |
| FM-6 | Balances view: "Jason owes Mei NT$1,240". Record a settlement with one tap. | P0 |
| FM-7 | Activity feed and push notification when another member adds/edits a shared transaction (configurable). | P1 |
| FM-8 | Allowance management for children: scheduled allowance, child sees own balance (Viewer + own personal ledger). | P2 |
| FM-9 | Guest members for trip ledgers (non-family, e.g. friends or roommates) with limited access. | P2 |
| FM-10 | Conflict-safe concurrent edits: last-writer-wins per field with an edit history visible on the transaction. | P0 |

### 6.5 Budgets & Goals

| ID | Requirement | Priority |
|----|-------------|----------|
| BG-1 | Monthly budget per ledger, total and per category. | P0 |
| BG-2 | Progress bar on home screen; alerts at 80 % and 100 % (configurable). | P0 |
| BG-3 | Budget rollover option (unused amount carries to next month). | P1 |
| BG-4 | Savings goals with target amount, deadline, and linked account; projected completion date. | P1 |
| BG-5 | Custom budget periods (weekly, payday-to-payday). | P2 |

### 6.6 Reports & Insights

| ID | Requirement | Priority |
|----|-------------|----------|
| RP-1 | Monthly overview: income, expense, net, comparison with previous month. | P0 |
| RP-2 | Category breakdown (donut + ranked list), drill down to transactions. | P0 |
| RP-3 | Trend chart over 6 / 12 months, per category. | P0 |
| RP-4 | Per-member spending within a shared ledger. | P1 |
| RP-5 | Calendar view of daily spending. | P1 |
| RP-6 | Automatic monthly summary push ("You spent 12 % less on dining this month"). | P1 |
| RP-7 | Year-in-review report. | P2 |

### 6.7 Data Management

| ID | Requirement | Priority |
|----|-------------|----------|
| DM-1 | Offline-first: every feature except invitations works offline; sync when online. | P0 |
| DM-2 | Cloud sync across the user's devices (requires account). | P0 |
| DM-3 | Export to CSV and Excel (per ledger, date range). | P0 |
| DM-4 | Import from CSV with column mapping; import presets for popular Taiwanese apps (e.g. CWMoney, Moneybook export formats). | P1 |
| DM-5 | Full account deletion with all data, self-service, completed within 30 days. | P0 |
| DM-6 | Local encrypted backup file (for users who never sign up). | P1 |

### 6.8 Settings & Personalization

| ID | Requirement | Priority |
|----|-------------|----------|
| ST-1 | Manage categories (add, rename, reorder, icon, color, merge, archive). | P0 |
| ST-2 | Dark mode, large-text support (respects system font scale). | P0 |
| ST-3 | Week start day, month start day (e.g. payday on the 5th). | P1 |
| ST-4 | Daily logging reminder at a chosen time. | P0 |

## 7. Key User Flows

### 7.1 Log an expense (target ≤ 5 s)

```
[Tap +] → [Keypad: 150] → [Tap "Lunch" chip] → [Save]
                 │
                 └─ optional: change ledger (Personal ▸ Household), account, date, note, photo
```

### 7.2 Log a shared expense and settle up

```
Jason: + → 2,480 → Groceries → Ledger: Household → Paid by: Jason → Split: Equal (Mei, Jason) → Save
Mei's app: notification "Jason added Groceries NT$2,480 to Household"
Household › Balances: "Mei owes Jason NT$1,240" → [Settle up] → choose account → Save
```

### 7.3 E-invoice import (P1)

```
Settings › E-invoice › Link carrier (barcode + verification code)
Daily sync → Inbox badge "6 new invoices"
Inbox: swipe right = accept with suggested category, tap = edit, swipe left = ignore
```

### 7.4 Home screen layout (wireframe)

```
┌──────────────────────────────────┐
│ November ▾        Personal ▾  🔔 │
│ Spent  NT$18,420 / 25,000        │
│ ███████████████░░░░░  74 %       │
├──────────────────────────────────┤
│ Today                            │
│  🍱 Lunch          -150   Cash   │
│  🚇 MRT            -35  EasyCard │
│ Yesterday                        │
│  🛒 Groceries   -2,480  Household│
│     Paid by Jason · split 2      │
├──────────────────────────────────┤
│  Home   Reports   (+)   Family  ⚙ │
└──────────────────────────────────┘
```

## 8. Non-Functional Requirements

| Area | Requirement |
|------|-------------|
| Performance | Cold start ≤ 2 s on a mid-range Android device (e.g. 4 GB RAM, 2022 chipset). Quick-add screen opens ≤ 300 ms. Lists of 10,000 transactions scroll at 60 fps. |
| Offline & sync | All writes succeed offline; sync completes ≤ 5 s after reconnect for ≤ 500 pending changes. No data loss on conflict. |
| Security | TLS 1.2+ in transit; AES-256 at rest on server; local database encrypted (SQLCipher or platform equivalent). Secrets in Android Keystore / iOS Keychain. |
| Privacy | Compliant with Taiwan Personal Data Protection Act (PDPA) and GDPR-level rights (access, export, deletion). No third-party ad SDKs. Analytics are anonymized and opt-out-able. |
| Reliability | Backend availability ≥ 99.9 %. Daily server backups, 30-day retention, tested restore. |
| Accessibility | Supports TalkBack / VoiceOver, dynamic font size up to 200 %, contrast ratio ≥ 4.5:1. |
| Localization | zh-TW and English at launch; all strings externalized; locale-aware number, date, and currency formatting (TWD has no decimals by default). |
| Money correctness | Amounts stored as integer minor units plus ISO 4217 currency code — never floating point. Split remainders allocated deterministically (e.g. extra NT$1 to the payer). |
| Testability | Every interactive element has a stable accessibility ID / resource-id so the app can be covered by Appium UI automation in CI. |
| Platform support | Android 10+ and iOS 16+. **[PM call]** |

## 9. Success Metrics

**North Star metric:** *Weekly Active Logging Households* — households (personal-only users count as a household of one) with ≥ 3 logging days in the past 7 days.

| Type | Metric | MVP target (90 days post-launch) |
|------|--------|-----------------------------------|
| Activation | % of new installs logging ≥ 3 transactions on day 1 | ≥ 50 % |
| Engagement | Median transactions logged per active user per week | ≥ 12 |
| Retention | D7 / D30 retention | ≥ 50 % / ≥ 35 % |
| Family adoption | % of signed-up users in a family group with ≥ 2 members | ≥ 40 % |
| Speed | Median quick-add duration (open → save) | ≤ 5 s |
| Quality | Crash-free sessions | ≥ 99.5 % |
| Satisfaction | Store rating | ≥ 4.5 |

## 10. Business Model **[PM call]**

Freemium, no ads.

| Tier | Includes |
|------|----------|
| **Free** | Personal ledger, unlimited transactions, 1 family group with up to 2 members, 1 shared ledger, basic reports, CSV export, cloud sync. |
| **Family Plus** (≈ NT$60 / month or NT$590 / year, covers the whole family group) | Up to 6 members, unlimited shared ledgers, e-invoice import, receipt photos beyond 100, advanced reports, multi-currency, budget rollover, priority support. |

Rationale: the core habit (logging) must stay free; the paid value is in deeper family collaboration and automation.

## 11. Release Plan

| Phase | Scope | Exit criteria |
|-------|-------|---------------|
| **Phase 0 — Discovery** (4 weeks) | 10–15 household interviews, competitor teardown, clickable prototype test of quick-add and settle-up flows. | Validated primary persona, quick-add ≤ 5 s in prototype tests. |
| **Phase 1 — MVP / closed beta** (≈ 12 weeks) | All **P0** requirements. Android first (larger share in Taiwan, existing Appium test infrastructure), iOS in parallel if using a cross-platform stack. | ≥ 200 beta households, crash-free ≥ 99 %, D7 ≥ 40 %. |
| **Phase 2 — Public launch v1.0** | P0 polish + Family Plus subscription + LINE login. | Store rating ≥ 4.3, payment flow live. |
| **Phase 3 — v1.x** | P1 items, led by e-invoice import (TX-8), widget (TX-7), smart category (TX-11), multi-currency (TX-6). | Metrics in §9 met. |
| **Phase 4 — Later** | P2 items: OCR, voice entry, allowance, guest members, year-in-review, web dashboard. | Driven by usage data and feedback. |

## 12. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Users abandon logging after the first week | High | Sub-5 s quick-add, widget, daily reminder, e-invoice automation, streak/positive feedback without guilt. |
| Sync conflicts corrupt shared ledgers | High | Per-field last-writer-wins with full edit history; extensive offline/concurrency test suite. |
| Family privacy concerns block adoption | High | Strict personal-by-default rule (§5); clear privacy explainer during family invite. |
| E-invoice platform API access / rate limits change | Medium | Treat as P1, not a launch dependency; abstract behind an import adapter; fall back to manual CSV import. |
| Security incident with financial data | Critical | Encryption at rest and in transit, external penetration test before launch, minimal data collection, bug-bounty program. |
| Crowded market (many local expense apps) | Medium | Differentiate on family + privacy + speed rather than feature count. |

## 13. Open Questions

1. Is Taiwan the right launch market, or should we go region-neutral from day one?
2. Should guest (non-family) members — roommates, trip friends — be in MVP? It widens the audience but complicates the family-group model.
3. Do family members need to see each other's *account balances* (net worth) or only shared-ledger spending?
4. Pricing validation: is a per-family subscription accepted, or do users expect a one-time purchase?
5. Should children (under 13) be allowed their own accounts? This has legal consent implications.
6. Cross-platform stack (Flutter / React Native / Kotlin Multiplatform) vs. native — to be decided in the technical design (`doc/tech/`).

## 14. Appendix

### 14.1 Default category set (expense)

Food (Breakfast, Lunch, Dinner, Snacks, Groceries) · Transport (MRT/Bus, Taxi, Fuel, Parking, HSR/Train) · Housing (Rent/Mortgage, Utilities, Internet, Management fee) · Shopping (Clothing, Electronics, Household items) · Health (Medical, Pharmacy, Insurance) · Education (Tuition, Books, Courses) · Entertainment (Movies, Games, Subscriptions, Travel) · Family (Kids, Pets, Gifts, Red envelopes 紅包, Parents' allowance) · Finance (Fees, Interest, Tax) · Other

### 14.2 Default category set (income)

Salary · Bonus · Side income · Investment income · Gifts / Red envelopes · Refunds · Other

### 14.3 Glossary

| Term | Meaning |
|------|---------|
| Carrier (載具) | Taiwan Ministry of Finance mobile barcode used to store cloud e-invoices. |
| Settlement | A transaction that records repayment between family members in a shared ledger. |
| Minor units | Smallest currency unit (e.g. cents); TWD is typically recorded in whole dollars. |
