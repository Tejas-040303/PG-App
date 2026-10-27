# Data model

> **Status: design — not implemented.** Phase 1 entities are the intended first migration. Phase 2–3 entities are a forward sketch to make sure Phase 1 does not paint us into a corner; they will be revisited before those phases start.

## Conventions

- Primary keys are UUIDs. They avoid leaking record counts in URLs. They are **not** a security control — authorization is.
- Tables and columns use `snake_case` in PostgreSQL (`@@map` / `@map` in Prisma); code uses `camelCase`.
- All timestamps are `timestamptz` in UTC. Calendar dates are `date`.
- Money is an integer number of paise plus a `currency` column (`'INR'`).
- Rows that form history (allocations, payments, audit) are never deleted or overwritten. State changes add or close rows.
- Every tenant-owned table has a non-null `organization_id`.

## Phase 1 entities

### `users` — a person's identity

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| email | text | Stored lower-cased and trimmed. **Unique.** |
| name | text | |
| password_hash | text | Argon2id encoded hash (algorithm and parameters are inside the string) |
| status | enum `ACTIVE`, `SUSPENDED` | Suspended users cannot sign in, and existing sessions stop working immediately. |
| created_at, updated_at | timestamptz | |

A user is identity only. What they may do lives in memberships, staff assignments, and residencies.

### `organizations` — the business that owns properties

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| name | text | |
| created_at, updated_at | timestamptz | |

### `org_memberships` — a user's role in one organization

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations |
| user_id | uuid | FK → users |
| role | enum `OWNER`, `MANAGER`, `MESS_STAFF`, `MAINTENANCE_STAFF` | |
| status | enum `ACTIVE`, `REVOKED` | Revocation keeps the row for history. |
| created_at, revoked_at | timestamptz | |

Invariants:
- At most one `ACTIVE` membership per (user, organization). Enforced with a partial unique index on `(organization_id, user_id) WHERE status = 'ACTIVE'`.
- An organization always has at least one `ACTIVE` `OWNER`. Enforced in the service layer (revoking the last owner is rejected).

Residents are **not** members. Resident access comes from a residency (Phase 3), because a resident's relationship to a property has dates, a bed, and money attached — a staff role does not.

### `sessions` — one signed-in browser

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK → users |
| token_hash | bytea | SHA-256 of the cookie token. **Unique.** The raw token is never stored. |
| created_at | timestamptz | |
| last_seen_at | timestamptz | Updated at most every few minutes to limit writes. |
| expires_at | timestamptz | Absolute expiry. |
| revoked_at | timestamptz, nullable | Set on logout. |

A database leak therefore does not hand out usable session cookies.

### `audit_logs` — who did what

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid, nullable | Null for pre-tenant events (for example a failed login). |
| actor_user_id | uuid, nullable | |
| action | text | For example `auth.login.succeeded`, `organization.updated`. |
| target_type, target_id | text, nullable | |
| metadata | jsonb | Never contains passwords, tokens, or full request bodies. |
| ip | inet, nullable | Recorded for authentication events only. Retention period to be decided. |
| created_at | timestamptz | |

Append-only from the application's point of view.

## Forward sketch (Phase 2–3)

```text
organizations ──< properties ──< rooms ──< beds
                     │
                     ├──< property_staff_assignments >── org_memberships
                     ├──< invitations
                     └──< residencies >── users
                              │
                              └──< bed_allocations >── beds
```

- **properties**: `organization_id`, name, address, `timezone` (default `Asia/Kolkata`).
- **rooms**: `property_id`, label, optional `block` and `floor` text, sharing type.
- **beds**: `room_id`, label, operational status (`AVAILABLE_FOR_USE`, `OUT_OF_SERVICE`). Whether a bed is *occupied* is derived from allocations, not stored on the bed.
- **property_staff_assignments**: narrows a MANAGER or staff membership to specific properties.
- **invitations**: `token_hash`, intended email, intended residency or role, `expires_at`, `accepted_at`. Single use.
- **residencies**: a person's tenancy at a property — `user_id`, `property_id`, `starts_on`, `ends_on`, status. Rent invoices (Phase 6) attach here.
- **bed_allocations**: which bed a residency occupies over which dates — `bed_id`, `residency_id`, `starts_on`, `ends_on` (null = open-ended), `status` (`ACTIVE`, `CANCELLED`).

### Preventing double allocation, including future reservations

Requirement: one bed must never be promised to two residencies for any overlapping night — even when one allocation starts next month.

A partial unique index on "the active allocation" cannot express that, because it has no notion of time. A PostgreSQL **exclusion constraint** on date ranges can:

```sql
-- Sketch only. Will be written and tested in the Phase 3 migration.
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bed_allocations
  ADD CONSTRAINT bed_allocations_no_overlap
  EXCLUDE USING gist (
    bed_id WITH =,
    daterange(starts_on, ends_on, '[)') WITH &&
  )
  WHERE (status <> 'CANCELLED');
```

- `'[)'` makes ranges half-open: someone moving out on the 5th and someone moving in on the 5th do not conflict.
- A null `ends_on` is an unbounded range, so an open-ended stay blocks every later allocation until it is given an end date.
- `btree_gist` is needed to combine `=` on `bed_id` with `&&` on the range inside one GiST index.
- A second constraint of the same shape on `residency_id` stops one resident holding two beds at once.
- A violation raises SQLSTATE `23P01`, which the API maps to `409 Conflict`. Under concurrent requests PostgreSQL guarantees at most one of the conflicting inserts commits.

Risks to verify in Phase 3:
- Prisma's schema language cannot express exclusion constraints, so the constraint lives in a hand-edited migration. We must confirm `prisma migrate dev` does not try to drop it as drift.
- The chosen managed PostgreSQL provider must allow the `btree_gist` extension.

A transfer is "close the current allocation + open a new one" in one transaction, on the same residency. History is preserved because no allocation row is ever deleted.
