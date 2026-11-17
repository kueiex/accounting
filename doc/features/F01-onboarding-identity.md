# F01 — Onboarding & Identity

| Item | Value |
|------|-------|
| PRD | §6.1 (ON-1 … ON-5) |
| Priority | P0 (LINE login P1) |
| Milestone | M1 (sign-in), M5 (demo mode) |
| Pages | `sign_in` (unauthenticated), `onboarding` (first run) |
| Tech refs | [02 § 3 Authentication](../tech/02-system-design.md#3-authentication--identity), [05 § 2.1](../tech/05-api-design.md#21-identity--onboarding-f01) |

## 1. Summary

A visitor signs in with one tap through an OIDC broker (Google, Apple, LINE, email code). On first sign-in Kinbook provisions a personal ledger, default categories, and a Cash account. A short skippable onboarding sets currency, language, accounts, and an optional monthly budget. A demo mode lets visitors try the app without an account.

## 2. User stories

| ID | As a … | I want … | So that … |
|----|--------|----------|-----------|
| US-01.1 | new visitor | to try the app with sample data without signing in | I can judge it before committing |
| US-01.2 | new user | to sign in with an account I already have (Google / Apple / LINE / email) | I don't create another password |
| US-01.3 | new user | to be ready to log an expense within 30 seconds of signing in | I don't give up during setup |
| US-01.4 | user | my language and currency chosen automatically and changeable later | the app feels local |
| US-01.5 | privacy-conscious user | the session to end automatically when idle | others using my phone can't see my finances |

## 3. Requirement breakdown

| ID | Sub-feature | Status | Priority | Acceptance criteria |
|----|-------------|--------|----------|---------------------|
| F01-1 | OIDC sign-in | Keep | P0 | **Given** a signed-out visitor **when** they tap "Sign in" **then** `st.login("kinbook")` redirects to the broker, and on return `st.user.is_logged_in` is true. **Given** the same `(iss, sub)` signs in again **then** no new user is created. |
| F01-2 | User provisioning | Keep | P0 | The first sign-in creates in **one transaction**: `users`, `auth_identities`, personal `ledgers` row, default categories (§[Appendix 14.1/14.2 of PRD](../plan/accounting-app-prd.md#14-appendix)), and a "Cash" account in the default currency. If any step fails, nothing is created. |
| F01-3 | Locale & currency defaults | Keep | P0 | `locale` comes from the ID-token `locale` claim (falls back to `zh-TW`). Currency is `TWD` for `zh-TW`, `USD` for `en`. Both are editable on the onboarding step 1. |
| F01-4 | Guided onboarding | Keep | P0 | Three steps (language & currency → accounts → monthly budget). Each is skippable. "Skip all" goes to the quick-add page. Completing it sets `users.onboarded_at`. Re-entry is not shown again. |
| F01-5 | Demo mode | Adapted (ON-1) | P0 | `/?demo=1` builds an in-memory container with ~60 sample transactions over 2 months. A banner reads "Demo — nothing is saved" and offers a sign-in button. No DB writes occur (asserted in test). |
| F01-6 | Idle timeout & sign-out | Adapted (ON-5) | P0 | `st.session_state.last_activity` is updated on every rerun. After 30 min idle the next rerun calls `st.logout()` and shows the sign-in page. A manual sign-out button is in Settings. |
| F01-7 | LINE login | Keep | P1 | Enabled as an identity provider in the broker. No app change. |
| F01-8 | Passkey / MFA | Adapted (ON-5) | P1 | Broker configuration. Users can enroll a passkey from the broker's account page (linked from Settings). |

## 4. Business rules

1. Kinbook stores no passwords. Identity is `(issuer, subject)` from the broker.
2. Email from the ID token is stored only if it is verified (`email_verified = true`).
3. A user can link several identities (e.g. Google + LINE) through the broker. The broker returns one stable `sub`, so Kinbook sees a single identity.
4. The personal ledger cannot be deleted or archived. It exists for the lifetime of the user.

## 5. UI (Streamlit)

```
sign_in                              onboarding (step 2 of 3)
┌───────────────────────────┐        ┌───────────────────────────┐
│        Kinbook            │        │ Your accounts        2/3  │
│ Know where your family's  │        │ ┌ Cash      NT$ [  2,000] │
│ money goes.               │        │ ├ Bank      NT$ [ 50,000] │
│ [ Sign in ]               │        │ └ + Add account           │
│ [ Try the demo ]          │        │ [Skip]          [Next ▸]  │
└───────────────────────────┘        └───────────────────────────┘
```

| Widget key | Element |
|------------|---------|
| `sign_in.login` | `st.button` → `st.login("kinbook")` |
| `sign_in.demo` | `st.button` → sets query param `demo=1` |
| `onboarding.locale`, `onboarding.currency` | `st.selectbox` |
| `onboarding.account.<n>.name/type/balance` | inputs in a `st.form` |
| `onboarding.budget` | `money_input` |
| `onboarding.skip`, `onboarding.next` | `st.button` |

## 6. Data & API

- Tables: `users`, `auth_identities`, `ledgers`, `categories`, `accounts`, `budgets`.
- Schema addition: `users.onboarded_at timestamptz NULL`.
- Use cases: `SignInUser`, `CompleteOnboarding`, `UpdateProfile`, `GetMe`.

## 7. Performance

- `SignInUser` for a returning user: 1 indexed lookup on `auth_identities (issuer, subject)` + `UPDATE last_login_at`. The resolved `actor_id` is cached in session state, so later reruns run **0** identity queries.
- Provisioning a new user inserts the default categories (~45 rows) with one multi-row `INSERT`.

## 8. Test checklist

- [ ] Unit: `SignInUser` creates once and is idempotent for repeated `(iss, sub)`.
- [ ] Unit: provisioning rolls back completely if category seeding fails.
- [ ] Integration: query budget for returning sign-in ≤ 2.
- [ ] AppTest: signed-out state renders only the sign-in page (`st.stop()` reached).
- [ ] AppTest: demo mode performs zero DB statements.
- [ ] Playwright: sign-in via test IdP → onboarding → skip → quick add visible in < 30 s.
