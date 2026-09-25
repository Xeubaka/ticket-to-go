# ADR 0003 — Testing strategy and tooling

**Status:** accepted · 2026-09-25

## Decision

- **Vitest everywhere.** NestJS 12 scaffolds with Vitest (and oxlint) instead of Jest, and the frontend
  uses Vitest too. That gives one runner and one mocking API across the repo. (The original plan said
  Jest; we followed the framework's current default.)
- **Backend:** one Vitest config with three projects: `unit` (mocked collaborators), `integration`
  (real services/repositories) and `e2e` (Supertest through `AppModule`). DB-backed projects start a
  throwaway Postgres with Testcontainers, apply the real migrations, and truncate between tests.
  Coverage is measured across all three combined.
- **Frontend:** RTL + user-event unit tests; feature "integration" tests against MSW with fixtures
  typed from the OpenAPI schema; Playwright for full-stack journeys on Chromium, Firefox and a mobile
  viewport, with axe checks (WCAG 2.2 AA, no serious/critical violations).
- **Playwright isolation:** API on 3101 + Next build in `.next-e2e` on 3100 + the `postgres-test`
  database, reset (guarded to `*_test` databases) and seeded on every run. Tests run serially because
  they share one database.
- Thresholds: 80 % lines/branches/functions/statements on both apps, enforced in CI.

## Notes (Windows)

- Don't `process.exit()` right after a `fetch` (a libuv `UV_HANDLE_CLOSING` assertion crashes the
  process); let scripts end naturally.
- Killing a shell on Windows can orphan its `node` child. Free ports 3100/3101 before re-running
  Playwright if a previous run was interrupted.
