# Service Layer Compliance Tracker

**Rule (ADR-007):** Route handlers MUST NOT contain SQL, business logic, or direct external
integration calls. All domain logic lives in a named service class under
`apps/api/src/services/`. Route files are thin HTTP adapters: parse input, call service,
return result.

## Status at a glance

| Bounded context | Route file(s) | Service class | Status |
|-----------------|--------------|---------------|--------|
| Jobs | `routes/v1/jobs.ts` | `services/job-service.ts` | ✅ Compliant |
| Inspection | `routes/v1/inspection.ts` | `services/inspection-service.ts` | ✅ Compliant |
| Artifacts | `routes/v1/artifacts.ts` | uses `@clark/storage` abstraction | ✅ Compliant |
| Events | `routes/v1/events.ts` | uses `@clark/events` EventStore directly | ⚠️ Low risk — read-only, no business logic |
| Identity / Auth | `routes/v1/auth.ts`, `plugins/auth.ts` | ❌ missing `IdentityService` | 🔴 Violations |
| Facilities | `routes/v1/facilities.ts`, `routes/v1/workstations.ts` | ❌ missing `FacilityService` | 🔴 Violations |
| Notes | `routes/v1/notes.ts` | ❌ missing `NoteService` | 🔴 Violations |
| Issues | `routes/v1/issues.ts` | ❌ missing `IssueService` | 🔴 Violations |
| Conversations | `routes/v1/conversations.ts` | ❌ missing `ConversationService` | 🔴 Violations |
| Presence / Shifts | `routes/v1/presence.ts`, `routes/v1/shifts.ts` | ❌ missing `PresenceService` | 🔴 Violations |
| Permissions | `routes/v1/permissions.ts` | ❌ missing `PermissionService` | 🔴 Violations |
| AI Review | `routes/v1/ai.ts` | partial — `@clark/ai` has `buildReviewStateUpdate()`, but DB write still in route | 🟡 Partial |

## Per-context detail files

- [identity.md](identity.md)
- [facilities.md](facilities.md)
- [jobs.md](jobs.md)
- [inspection.md](inspection.md)
- [notes.md](notes.md)
- [issues.md](issues.md)
- [conversations.md](conversations.md)
- [presence-shifts.md](presence-shifts.md)
- [permissions.md](permissions.md)
- [ai-review.md](ai-review.md)
- [artifacts-events.md](artifacts-events.md)

## How to use this tracker

When you (or another reviewer) extract a route's logic into a service class:
1. Create `apps/api/src/services/<context>-service.ts`
2. Rewrite the route file as a thin adapter (no SQL imports, no `@clark/db` import)
3. Update the relevant `.md` file in this directory to mark it compliant
4. Update the table above
