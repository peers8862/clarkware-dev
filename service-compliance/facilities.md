# Facilities / Workstations — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/facility-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/facilities.ts`
Direct SQL in route handlers (exact violations to verify on read):
- `GET /v1/facilities` — queries `facilities` table
- `POST /v1/facilities` — inserts into `facilities`
- `GET /v1/facilities/:id` — queries by ID
- Any zone or workstation sub-routes

### `apps/api/src/routes/v1/workstations.ts`
Direct SQL in route handler:
- `GET /v1/workstations` — calls `query()` from `@clark/db` directly inside handler

## What FacilityService should own
- Data: `facilities`, `zones`, `workstations`
- No domain events required for Phase 1 (no downstream CFX messages for facility CRUD)
- Methods: `listFacilities()`, `getFacility(id)`, `createFacility(actor, input)`,
  `listWorkstations(facilityId?)`, `getWorkstation(id)`

## Migration notes
Low priority — facilities/workstations are configuration data with no CFX implications.
No outbox writes needed. Primary motivation for refactor is consistency with ADR-007, not
risk reduction.
