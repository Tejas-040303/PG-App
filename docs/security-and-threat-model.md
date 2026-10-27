# Security and threat model

> **Status: design — not implemented.** Having authentication does not make the application secure. This document lists what we defend against, how, and what is knowingly left open.

## What we protect

- Credentials and sessions.
- Tenant boundaries: one PG business must never see or change another's data.
- Resident personal data (names, contact details, later rent and maintenance history).
- Financial records (Phase 6).

## Who we defend against

| Actor | Example |
|---|---|
| Anonymous internet user | Credential stuffing against `/auth/login` |
| Signed-in user of another tenant | Changes `orgId` in a URL to read a competitor's data |
| Departed or demoted staff member | Keeps using an old browser session after access is revoked |
| Malicious website | Makes the victim's browser send a state-changing request (CSRF) |
| Anyone reading the public repository | Finds a committed secret |

## Controls (Phase 1)

| Area | Control |
|---|---|
| Passwords | Argon2id (fallback: scrypt from `node:crypto` if the native module causes problems on Windows). Length 12–128, no composition rules. Breached-password check planned. |
| Sessions | 256-bit random token in the cookie; only its SHA-256 hash is stored. Cookie is `HttpOnly`, `SameSite=Lax`, `Path=/`; in production also `Secure` with the `__Host-` prefix. Absolute lifetime 7 days, idle timeout 48 hours. A new session is issued on every login. Logout revokes the row. |
| Revocation | Suspension and membership revocation are checked against the database on every request, so they take effect immediately. |
| CSRF | `SameSite=Lax` plus an `Origin` allowlist check on every state-changing request, plus JSON-only request bodies. |
| Brute force | Per-IP rate limit on `register` and `login`. Deliberately **no** account lockout, because lockout lets an attacker lock a real owner out. |
| User enumeration | Login returns the same error for unknown email and wrong password, and runs a dummy hash for unknown emails so response time does not differ. Registration does reveal that an email is taken (`409`) — accepted until email verification exists. |
| Broken access control | Tenant guard on every organization-scoped route; negative tests for every endpoint (see [permissions.md](permissions.md)). |
| Injection | Prisma parameterized queries only. Raw SQL only through tagged templates, never string concatenation. |
| XSS | React escaping by default; no `dangerouslySetInnerHTML` on user content; Content-Security-Policy on the web app; session cookie unreadable by scripts. |
| Errors and logs | Clients get a generic message plus a request ID. Logs redact `cookie`, `authorization`, `password`, and token fields, and never log auth request bodies. |
| Secrets | Read from environment variables, validated at startup; the app refuses to start with missing or malformed config. Only `.env.example` with placeholders is committed. |

## Known gaps after Phase 1 — accepted for now

| Gap | Why deferred | Planned |
|---|---|---|
| No password reset | Needs an email provider (external service, needs approval) | Before pilot onboarding |
| No email verification | Same | Before pilot onboarding |
| No MFA | Pilot scale; adds friction | Re-evaluate after pilot |
| Rate limiter state is in memory | Correct only with a single API instance | Shared store when hosting is chosen |
| Audit IP retention undefined | Needs a privacy decision | Before pilot |
