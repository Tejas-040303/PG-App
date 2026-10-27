# API contract

> **Status: design — not implemented.** This is the Phase 1 contract the implementation will be tested against.

## Conventions

- Base path: `/api/v1`. JSON request and response bodies only; other content types are rejected with `415`.
- Authentication: session cookie set by `register` and `login`.
- State-changing requests must carry an `Origin` header matching the app's origin, or they are rejected with `403 ORIGIN_REJECTED`.
- Successful responses return the resource itself. List endpoints (from Phase 2) return `{ "data": [...], "nextCursor": "..." | null }` with a server-enforced maximum page size.
- Errors always use one envelope:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Human-readable summary",
    "details": [{ "path": "email", "message": "Invalid email" }],
    "requestId": "a1b2c3"
  }
}
```

`details` appears only for validation errors. Stack traces and database messages never reach the client.

| Status | Used for |
|---|---|
| 400 `VALIDATION_FAILED` | Malformed or invalid input |
| 401 `UNAUTHENTICATED` / `INVALID_CREDENTIALS` | No valid session / wrong email or password |
| 403 `FORBIDDEN` / `ORIGIN_REJECTED` / `ACCOUNT_SUSPENDED` | Authenticated but not allowed |
| 404 `NOT_FOUND` | Missing, or exists but the caller is not a member |
| 409 `CONFLICT` / `EMAIL_TAKEN` | Uniqueness or state conflict |
| 415 `UNSUPPORTED_MEDIA_TYPE` | Non-JSON body |
| 429 `RATE_LIMITED` | Too many attempts |

## Endpoints (Phase 1)

### `GET /api/v1/health`
Liveness. Always `200 {"status":"ok"}` if the process is serving. Does not touch the database.

### `GET /api/v1/health/ready`
Readiness. `200` if a trivial database query succeeds, otherwise `503`.

### `POST /api/v1/auth/register`
Creates an owner account, their organization, and an OWNER membership in **one transaction**, then signs them in.

Request:
```json
{ "name": "Asha Patil", "email": "asha@example.com", "password": "at least 12 chars", "organizationName": "Patil PG Services" }
```
- `email`: valid address; stored lower-cased and trimmed.
- `password`: 12–128 characters. No composition rules.

`201`: `{ "user": UserDto, "organization": OrganizationDto }` plus `Set-Cookie`.
Errors: `400`, `403 ORIGIN_REJECTED`, `409 EMAIL_TAKEN`, `429`.
Audit: `user.registered`.

Two simultaneous registrations with the same email must produce exactly one `201` and one `409` — the unique index decides, not a prior "does it exist?" read.

### `POST /api/v1/auth/login`
Request: `{ "email": "...", "password": "..." }`

`200`: `{ "user": UserDto }` plus a **new** session cookie (any previous session cookie in this browser is replaced, preventing session fixation).
Errors: `401 INVALID_CREDENTIALS` — identical for an unknown email and a wrong password; `403 ACCOUNT_SUSPENDED` — only after a correct password; `429`.
Audit: `auth.login.succeeded` / `auth.login.failed`.

### `POST /api/v1/auth/logout`
Revokes the current session and clears the cookie. `204`, also when no session exists (idempotent).
Audit: `auth.logout`.

### `GET /api/v1/auth/me`
`200`:
```json
{
  "user": { "id": "...", "name": "...", "email": "..." },
  "memberships": [{ "organizationId": "...", "organizationName": "...", "role": "OWNER" }]
}
```
Only `ACTIVE` memberships are listed. `401` without a valid session.

### `GET /api/v1/organizations/:orgId`
`200 OrganizationDto` for any active member. `404` for non-members, revoked members, and unknown IDs alike.

### `PATCH /api/v1/organizations/:orgId`
Request: `{ "name": "New name" }`. OWNER only.
`200 OrganizationDto`. Errors: `400`, `403` (member without OWNER role), `404` (non-member).
Audit: `organization.updated` with the old and new name.

## DTOs

```ts
type UserDto = { id: string; name: string; email: string };
type OrganizationDto = { id: string; name: string; createdAt: string /* ISO 8601 UTC */ };
```

`password_hash`, session data, and user status are never returned.
