# Inspection — Service Layer Compliance

**Status: ✅ Compliant**

## Service class
`apps/api/src/services/inspection-service.ts` — fully implemented

## Route file
`apps/api/src/routes/v1/inspection.ts` — thin adapter, no SQL, no business logic

## What the service owns
- Data: `inspection_steps`, `inspection_results`, `defect_records`
- Reads (no write): `jobs` (for context/permission checks only)
- Criteria: `@clark/ipc-criteria` (in-process, no DB)
- Domain events: `inspection.point.logged`, `inspection.step.completed`
- CFX outbox: `NonConformanceCreated` (on Fail), `InspectionCompleted` (on step complete)

## Endpoints
| Method | Path | Service method |
|--------|------|----------------|
| GET | `/inspection/criteria` | `getCriteriaForStep()` |
| GET | `/inspection/criteria/:id` | `getCriterion()` |
| POST | `/inspection/steps` | `createStep()` |
| GET | `/inspection/steps/job/:jobId` | `getStepsForJob()` |
| POST | `/inspection/steps/:stepId/complete` | `completeStep()` |
| POST | `/inspection/results` | `logInspectionPoint()` |
| GET | `/inspection/results/step/:stepId` | `getResultsForStep()` |
| GET | `/inspection/defects/job/:jobId` | `getDefectsForJob()` |
| POST | `/inspection/defects/:defectId/disposition` | `dispositionDefect()` |
