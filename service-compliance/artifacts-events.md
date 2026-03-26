# Artifacts / Events — Service Layer Compliance

## Artifacts

**Status: ✅ Compliant**

`apps/api/src/routes/v1/artifacts.ts` uses `@clark/storage` (presigned URL generation)
and has no direct SQL. No service class extraction needed — the storage package is the
correct abstraction boundary here.

If artifact metadata (size, checksum, mime type) needs to be persisted to the `artifacts`
table after upload, that should go in an `ArtifactService`. Not yet implemented — Phase 1
artifact handling is presigned URL only (client uploads directly to MinIO).

## Events

**Status: ⚠️ Low risk — acceptable for now**

`apps/api/src/routes/v1/events.ts` instantiates `EventStore` from `@clark/events` and calls
it directly from the route handler. This is a read-only query against immutable data with
no business logic. The event store is itself an abstraction over the `events` table.

This is acceptable but could be improved by wrapping in an `EventQueryService` if query
complexity grows (e.g. cross-stream correlation, fan-out, projection). Not a priority.
