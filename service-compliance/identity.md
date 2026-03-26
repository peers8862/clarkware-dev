# Identity / Auth — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/identity-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/auth.ts`
Direct SQL in route handlers:
- `POST /v1/auth/login` — queries `persons` table for password verification, inserts `refresh_tokens`
- `POST /v1/auth/refresh` — queries `refresh_tokens`, updates token
- `POST /v1/auth/logout` — updates `refresh_tokens` (revoke)

### `apps/api/src/plugins/auth.ts`
Direct SQL in Fastify plugin (request-scoped auth):
- Queries `actors` + `permission_grants` tables to build `request.actor`
- Should delegate to `IdentityService.resolveActor(actorId)` or similar

## What IdentityService should own
- Data: `actors`, `persons`, `refresh_tokens`
- Reads: `permission_grants` (PermissionService owns writes)
- Methods: `login(username, password)`, `refreshToken(token)`, `revokeToken(token)`, `resolveActor(actorId)`
- No domain events needed (auth events are operational logs, not domain events)

## Migration notes
`@clark/identity` package already exists — check what it exposes before writing new code.
The Fastify auth plugin currently attaches `request.actor` directly; this pattern is fine to
keep, but the SQL inside it should move to `IdentityService.resolveActor()`.
