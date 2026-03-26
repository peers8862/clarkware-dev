# Jobs — Service Layer Compliance

**Status: ✅ Compliant**

## Service class
`apps/api/src/services/job-service.ts` — fully implemented

## Route file
`apps/api/src/routes/v1/jobs.ts` — thin adapter, no SQL, no business logic

## What the service owns
- Data: `jobs`, `tasks` tables
- Domain events: `job.created`, `job.started`, `job.paused`, `job.resumed`, `job.completed`, `job.voided`, `job.reopened`
- CFX outbox: `WorkOrderStarted`, `WorkOrderCompleted`

## Known gap
`broadcastEvent()` (WebSocket push to connected clients) is called from `jobs.ts` in the old
prototype but is absent from `JobService`. Real-time broadcast needs to be re-wired — either
the service emits a domain event that a subscriber broadcasts, or the route calls broadcast
after the service call. For now the route may call `broadcastEvent` after `jobService.startJob()`
without this counting as a compliance violation (it is a notification side-effect, not
business logic or SQL).
