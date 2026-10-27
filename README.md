# StaySphere

A PG, hostel, and co-living operations platform, starting in Pune, India.

**Status:** pre-alpha. No application code yet. Design work is tracked in pull requests against `main`.

## Planned scope (first releases)

- Property, room, and bed setup with occupancy tracking
- Resident invitations and bed allocation (with future reservations)
- Rent invoices and payment records
- Maintenance requests, announcements, and meal planning — ordering to be decided by owner interviews

Nothing in this list is implemented yet.

## Design documents

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Stack, repository layout, request lifecycle, decision log |
| [docs/data-model.md](docs/data-model.md) | Entities, invariants, bed-allocation constraint |
| [docs/permissions.md](docs/permissions.md) | Roles, tenant-isolation rules, permission matrix |
| [docs/api.md](docs/api.md) | Phase 1 API contract and error format |
| [docs/security-and-threat-model.md](docs/security-and-threat-model.md) | Threats, controls, accepted gaps |
| [docs/testing.md](docs/testing.md) | Test strategy and Phase 1 test list |
| [docs/business-validation.md](docs/business-validation.md) | Hypotheses, interview plan, metric definitions |
