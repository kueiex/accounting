# Kinbook — Technical Documentation

These documents describe how Kinbook (see the product requirements document in [`../plan/accounting-app-prd.md`](../plan/accounting-app-prd.md)) is built with **Python + Streamlit**. The feature breakdown lives in [`../features/`](../features/README.md).

| # | Document | What it covers |
|---|----------|----------------|
| 01 | [Architecture](01-architecture.md) | Tech stack, clean-architecture layers, dependency rule, source tree, architecture decision records |
| 02 | [System design](02-system-design.md) | Runtime components, deployment, auth, Streamlit session/caching model, background worker, security, observability, testing |
| 03 | [Data flow](03-dataflow.md) | End-to-end sequence diagrams for the main read and write paths |
| 04 | [Database schema](04-database-schema.md) | ERD, table definitions, constraints, indexes, money and split rules |
| 05 | [API design](05-api-design.md) | Application-layer API (commands and queries), external integration contracts, future REST API |
| 06 | [Query performance](06-query-performance.md) | N+1 prevention rules, query budgets, caching, indexing, enforcement in CI |

## Platform decision and its impact on the PRD

The PRD was written for native mobile apps. The chosen stack, **Streamlit**, is a server-rendered **responsive web app**. This changes several requirements. The full list is in [`../features/README.md` § Platform adaptations](../features/README.md#platform-adaptations-streamlit). In summary:

- **Online-only.** Offline-first (DM-1), home-screen widgets (TX-7), and OS push notifications are not possible. They are replaced by in-app and email notifications and a bookmarkable quick-add URL.
- **Sign-in required for persistent data.** Local mode without sign-up (ON-1) becomes a non-persistent demo mode. The app PIN and biometric lock (ON-5) become identity-provider sessions with passkey support.
- **UI automation uses Playwright and Streamlit `AppTest`**, not Appium. The requirement for stable element IDs still holds: every widget has a stable `key`.

If native mobile apps are needed later, the clean-architecture core (domain and application layers) is reused behind a REST adapter. See [05 § Future REST API](05-api-design.md#4-future-rest-api-phase-4).
