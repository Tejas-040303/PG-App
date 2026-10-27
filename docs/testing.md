# Testing

> **Status: design — not implemented.** No tests exist yet.

## Principles

- Tests ship with the feature, in the same pull request.
- Integration tests run against a **real PostgreSQL database**, not mocks. Constraints, transactions, and concurrency behaviour only exist in the real database.
- A passing build is not proof that a workflow works or is secure; each report says exactly what was tested and what was not.

## Layers

| Layer | Tool | Covers |
|---|---|---|
| Unit | Vitest | Pure rules: validation schemas, state transitions, money arithmetic, permission decisions |
| API integration | Vitest + Supertest + PostgreSQL | Endpoints end to end through middleware, the database, and the error handler |
| Tenant isolation | Same as above | User from tenant B attempts to read or change tenant A's data by changing identifiers |
| Concurrency | Same as above | N parallel requests racing for one unique resource; exactly one must win |
| End to end | Playwright | Real browser through Next.js and the API (added once the web app has a workflow worth testing) |

## Test database safety

- Tests use a dedicated database whose name must end in `_test`. Test setup **refuses to run** against any other database name, so a mistyped `DATABASE_URL` cannot truncate real data.
- Migrations are applied with `prisma migrate deploy`. The suite never runs `migrate reset` or `migrate dev`.
- Tables are truncated between tests.

## Phase 1 test list

- Config: missing or invalid environment variables stop startup with a clear message.
- Register: success; creates user, organization, and OWNER membership; cookie flags; duplicate email → 409; invalid input → 400.
- Register concurrency: two parallel registrations with one email → exactly one 201.
- Login: success issues a new session; wrong password and unknown email give identical 401s; suspended user → 403.
- Sessions: expired session → 401; after logout the old cookie → 401; suspending a user kills their existing session.
- CSRF: state-changing request without `Origin` or with a foreign `Origin` → 403.
- Rate limit: repeated login attempts → 429.
- Tenant isolation: non-member GET or PATCH of another organization → 404; revoked member → 404; MANAGER PATCH → 403.
- Audit: register, login, logout, and organization update each write the expected audit row with no secrets in it.

## Continuous integration (planned)

GitHub Actions with a PostgreSQL service container: `npm ci` → build contracts → `prisma migrate deploy` → lint → typecheck → test → build.
