# Issues — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/issue-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/issues.ts`
Direct SQL in route handlers:
- `GET /v1/issues` — queries `issues` table
- `POST /v1/issues` — inserts new issue
- `PATCH /v1/issues/:id` — updates status/resolution
- Any escalation path

## What IssueService should own
- Data: `issues` table
- Domain events: `issue.opened`, `issue.escalated`, `issue.resolved`, `issue.closed`
- No CFX messages in Phase 1 (CFX NonConformance is covered by InspectionService defect flow)
- Methods: `listIssues(actor, filters)`, `getIssue(actor, id)`, `openIssue(actor, input)`,
  `escalateIssue(actor, id, escalateToActorId)`, `resolveIssue(actor, id, resolution)`,
  `closeIssue(actor, id)`

## Migration notes
Issues are referenced by `defect_records` via `job_id` but defect records are owned by
InspectionService. IssueService should NOT query `defect_records` — if a defect needs an
issue, InspectionService passes the issue ID at creation time (or a future event triggers it).
