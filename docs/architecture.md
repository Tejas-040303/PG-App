# Architecture

> **Status: design — not implemented.** Nothing described here exists in code yet. Sections will be updated to "implemented" as Phase 1 lands.

## Constraints that shape the design

- One developer, Windows (PowerShell + VS Code) as the primary dev machine.
- First users: a pilot of roughly 2–3 PG properties in Pune. Scale is not the problem; correctness, tenant isolation, and speed of iteration are.
- Hosting and budget are undecided. The design must not lock us into a specific provider.
- The public repository is visible to anyone, so no secret may ever be committed.

## Stack

| Layer | Choice | Why |
|---|---|---|
| API | Node.js + Express + TypeScript | Business logic stays framework-independent and testable with Supertest; a versioned REST API can later serve a mobile app. |
| Web | Next.js (App Router) + React + TypeScript | The authenticated app does not need SSR, but the later public marketplace benefits from it; choosing Next.js now avoids a migration later. |
| Styling | Tailwind CSS + daisyUI | Consistent components without building a design system first. |
| Validation | Zod, shared through a `contracts` package | One schema defines the request shape for both client-side forms and server-side validation. |
| Database | PostgreSQL | Transactions, constraints, partial indexes, and exclusion constraints (needed for bed allocation). |
| ORM | Prisma | Typed queries and migration history. Features Prisma cannot express are added as hand-written SQL in migrations. |
| Tests | Vitest + Supertest + real PostgreSQL; Playwright for end-to-end | See [testing.md](testing.md). |

Exact package versions are pinned at install time, not in this document.

## Repository layout (planned)

```text
apps/
  api/                 Express API (/api/v1)
    prisma/            schema.prisma + migrations
    src/
      config/          environment validation (fails fast at startup)
      http/            app factory, middleware, error handler
      modules/
        identity/      users, passwords, sessions, auth routes
        organizations/ organizations, memberships, tenant guard
        audit/         audit log writer
  web/                 Next.js app
packages/
  contracts/           Zod request schemas + response DTO types shared by api and web
docs/
```

npm workspaces tie these together. `contracts` is compiled to `dist/` and consumed like any other package, which keeps Next.js, tsx, and tsc resolution boring and predictable.

## Deployment shape: same origin

```text
Browser ──► https://app.example ──► Next.js
                 │
                 └── /api/*  ──(rewrite/proxy)──► Express API ──► PostgreSQL
```

The browser only ever talks to one origin. Consequences:

- Session cookies are first-party, so `SameSite=Lax` works and no CORS configuration is needed.
- CSRF defence is still required (see [security-and-threat-model.md](security-and-threat-model.md)); same-origin removes CORS complexity, not CSRF risk.
- The API remains independently deployable and testable; same-origin is a routing decision, not a coupling.

## Request lifecycle (API)

```text
request
  → security headers + request ID
  → Origin check (POST/PUT/PATCH/DELETE only)
  → session resolution (cookie → session row → user; rejects expired/revoked/suspended)
  → route
  → Zod validation of params/body/query
  → tenant guard (loads caller's ACTIVE membership for the organization in the URL)
  → role check
  → service function (database transaction for multi-row changes)
  → audit log entry (same transaction where the change is sensitive)
  → response DTO (never a raw database row)
```

Errors at any stage go to one error handler that returns the standard error envelope (see [api.md](api.md)) and never exposes stack traces.

## Module rules

1. A module owns its tables. Other modules call its service functions; they do not query its tables directly.
2. Every service function that touches tenant data takes an explicit tenant context (`{ userId, organizationId, role }`) as an argument. There is no global "current tenant" variable that a background job could forget to set.
3. Routes are thin: parse, authorize, call a service, map to a DTO.
4. Cross-module workflows (for example "accept invitation → create residency → allocate bed") live in the module that owns the workflow's outcome and run in one transaction.

## Decision log

Each decision lists what would make us revisit it.

| # | Decision | Alternatives considered | Revisit when |
|---|---|---|---|
| D1 | Modular monolith (one API process) | Microservices | A module has a genuinely different scaling or deployment need. |
| D2 | Separate Express API + Next.js web in npm workspaces | Next.js route handlers only; Vite SPA | Operating two processes becomes a real cost problem. |
| D3 | Same-origin deployment via Next.js rewrites | Separate API subdomain with CORS | A native mobile client or third-party API consumers appear. |
| D4 | Opaque, database-backed sessions | JWT access/refresh tokens | Request volume makes one session lookup per request measurably expensive. |
| D5 | Shared schema; `organization_id` on tenant rows; tenant guard in application code | Schema-per-tenant; PostgreSQL row-level security | Evaluate RLS as defence-in-depth once Phase 3 is stable. |
| D6 | Every owner gets an Organization, even a single-property owner | Owner-without-organization path | Never expected; this keeps one ownership path. |
| D7 | Bed allocations use date ranges protected by a PostgreSQL exclusion constraint (future reservations are in scope) | Partial unique index on "active" allocation | — (see [data-model.md](data-model.md)) |
| D8 | Money stored as integer paise plus an explicit currency code | `numeric(12,2)` | Multi-currency or fractional-paise pricing is needed. |
| D9 | Timestamps in UTC (`timestamptz`); each property has an IANA timezone (default `Asia/Kolkata`); calendar dates (meal day, move-in date) stored as `date` | Store local times | — |
| D10 | Owners and staff sign in with email + password. Residents join through a single-use invite link and set a password. Phone OTP deferred. | Phone OTP for everyone | Pilot residents struggle with passwords. SMS in India needs a provider and DLT registration — verify current rules before choosing. |
| D11 | Property → Room → Bed only; rooms carry optional `block` and `floor` text labels | Building and Floor entities | A pilot property genuinely needs per-building configuration. |
| D12 | Hosting deferred. No in-memory state except the auth rate limiter, which is documented as single-instance only. | — | Hosting is chosen (Phase 10). |
