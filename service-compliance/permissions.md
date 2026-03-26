# Permissions — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/permission-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/permissions.ts`
Direct SQL in route handlers (3 confirmed `query()` calls):
- `GET /v1/permissions` — queries `permission_grants` table
- `POST /v1/permissions` — inserts grant record
- `DELETE /v1/permissions/:id` — revokes grant (updates `revoked_at`)

## What PermissionService should own
- Data: `permission_grants`
- Domain events: `permission.granted`, `permission.revoked` (compliance audit trail)
- No CFX messages needed
- Methods: `listGrants(actor, scope)`, `grantPermission(actor, input)`,
  `revokePermission(actor, grantId)`

## Migration notes
`@clark/identity` package has a `can()` helper that reads `request.actor.permissionGrants`
(pre-fetched at auth time). That pattern is fine to keep. PermissionService is for the
management endpoints only (CRUD on grants), not the request-level enforcement.
