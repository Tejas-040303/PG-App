# Permissions and tenant isolation

> **Status: design — not implemented.**

## Where access comes from

```text
User (identity)
 ├── platform role           Phase 8   SUPER_ADMIN, PLATFORM_SUPPORT — cross-tenant, every use audited
 ├── org_memberships         Phase 1   OWNER, MANAGER, MESS_STAFF, MAINTENANCE_STAFF — per organization
 │     └── property_staff_assignments  Phase 2   narrows a membership to specific properties
 └── residencies             Phase 3   resident access to one property, for a date range
```

The same person can hold several of these at once — for example a resident at one PG and a manager at another. Access is always evaluated for the specific organization or property in the request, never from a single global role.

## Tenant isolation rules

1. The server decides scope. An `organizationId` or `propertyId` in a URL or body is a *lookup key*; the server then checks that the caller has an active relationship to it.
2. Every query on tenant data filters by the organization resolved from the caller's membership, not by a value copied from the request.
3. When linking two records (for example a bed to a residency), both must belong to the same organization. This is checked in the service layer and, where practical, by composite foreign keys.
4. Not a member → `404 Not Found`. Member without the needed role → `403 Forbidden`. Returning 404 to outsiders avoids confirming that another tenant's record exists.
5. Membership revocation and user suspension take effect on the very next request. Session resolution and the tenant guard read the database on every request; nothing is cached in the cookie.
6. Exports, reports, file downloads, notifications, and background jobs follow the same rules. Jobs receive an explicit tenant context.
7. Every rule above gets a negative test: a user from tenant B changes the identifier and must be denied.

## Phase 1 permission matrix

| Endpoint | Anonymous | Signed in, not a member | MAINTENANCE / MESS staff | MANAGER | OWNER |
|---|---|---|---|---|---|
| `POST /auth/register` | ✅ creates a new user, organization, and OWNER membership | n/a | n/a | n/a | n/a |
| `POST /auth/login` | ✅ | ✅ (replaces the current session) | ✅ | ✅ | ✅ |
| `POST /auth/logout` | ✅ (no-op) | ✅ | ✅ | ✅ | ✅ |
| `GET /auth/me` | 401 | ✅ | ✅ | ✅ | ✅ |
| `GET /organizations/:orgId` | 401 | 404 | ✅ | ✅ | ✅ |
| `PATCH /organizations/:orgId` | 401 | 404 | 403 | 403 | ✅ |

In Phase 1 only owners are created (through registration). Other roles arrive with invitations in Phase 2–3; the enum exists now so the tenant guard and tests are written against the final shape.
