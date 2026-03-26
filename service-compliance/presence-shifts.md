# Presence / Shifts — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/presence-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/presence.ts`
Direct SQL in route handlers:
- `GET /v1/presence` — queries `presence_states` by workstation
- `PUT /v1/presence` — upserts into `presence_states`

### `apps/api/src/routes/v1/shifts.ts`
Direct SQL in route handlers:
- `GET /v1/shifts` — queries `shifts`
- `POST /v1/shifts` — inserts shift record (clock-in)
- `PATCH /v1/shifts/:id` — updates `ended_at` (clock-out), links handoff note

## What PresenceService should own
- Data: `presence_states`, `shifts`
- Domain events: `shift.started`, `shift.ended` (for audit trail)
- CFX outbox: `OperatorActivity` (IPC-CFX v2.0) — publish on shift start/end
- Methods: `getPresence(workstationId)`, `setPresence(actor, workstationId, state)`,
  `startShift(actor, input)`, `endShift(actor, shiftId, handoffNote?)`

## Migration notes
`OperatorActivity` CFX message (v2.0) is not yet wired anywhere. This service is where
it belongs — `startShift()` and `endShift()` should write to `cfx_outbox` in the same
transaction as the `shifts` insert/update.
