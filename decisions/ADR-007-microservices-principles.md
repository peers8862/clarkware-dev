# ADR-007: Microservices Architecture Principles for Clarkware

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

An architectural review of the P1V1 prototype found significant violations of microservices principles across the codebase: business logic embedded in route handlers (transaction script anti-pattern), a shared database accessed by all contexts without ownership boundaries, the AI package writing directly to tables owned by other contexts, and external integrations (CFX/AMQP) called synchronously from HTTP request handlers bypassing the durable outbox.

These violations must be resolved before the platform scales to additional bounded contexts (Inspection, Firmware, AI agents) — otherwise each new feature deepens the coupling and makes eventual service extraction or independent testing impossible.

---

## Decision

### 1. Architecture style: Modular Monolith → Microservices migration path

Phase 1 deploys as a modular monolith — one Fastify API process, one PostgreSQL database, one Theia IPE process — but code is structured as if each bounded context is an independently deployable service. This means:

- Each bounded context has a dedicated service class that encapsulates all business logic and data access for that context
- Contexts communicate only via domain events or defined service interfaces — never via direct imports of another context's internals
- Each context owns a set of database tables; no other context writes to them

When the platform requires independent scaling or deployment, each service class can be extracted into its own process with minimal refactoring.

### 2. Layered architecture per bounded context

Every bounded context follows this layer structure:

```
HTTP Route Handler         — thin adapter: validates input, calls service, returns response
    ↓
Domain Service Class       — orchestrates business logic, owns state transitions
    ↓
Domain Event Append        — every state change produces an event in the event store
    ↓
Outbox Write (same txn)    — any external integration payload written transactionally
    ↓
Repository / Query         — data access (no raw SQL in service or route layer)
    ↓
PostgreSQL
```

External integrations (CFX/AMQP, AI, email) are never called synchronously from inside a database transaction. They are always triggered by the outbox pattern or an event handler after the transaction commits.

### 3. The Outbox Pattern is mandatory for all external integrations

Any message that must leave the process boundary (CFX/AMQP, email, webhook, AI call) MUST be written to an outbox table in the same PostgreSQL transaction as the domain state change that triggers it. A background flusher delivers the outbox messages asynchronously. This guarantees:

- A job can start even if RabbitMQ is down
- CFX messages are never lost even if the process crashes between publishing and acknowledgement
- The HTTP response is not held waiting for external systems

**Outbox tables:**
- `cfx_outbox` — IPC-CFX AMQP messages
- (future) `notification_outbox` — email/SMS/push notifications
- (future) `webhook_outbox` — customer webhook deliveries

### 4. Data ownership rules

Each bounded context owns a set of tables. Ownership means:
- Only the owning context's service class writes to those tables
- Other contexts may read via a query/repository (read model), but never write
- If context B needs context A's data, context B queries A's read model — it does not import A's service

**Ownership map (Phase 1):**

| Context | Owns Tables |
|---------|------------|
| Identity | `actors`, `persons`, `agents`, `refresh_tokens`, `permission_grants` |
| Jobs | `jobs`, `tasks` |
| Issues | `issues` |
| Notes | `notes` |
| Conversations | `conversations`, `messages`, `conversation_participants` |
| Inspection | `inspection_steps`, `inspection_results`, `defect_records` |
| Firmware | `firmware_records` (new) |
| Shifts | `shifts` |
| Presence | `presence_states` |
| Artifacts | `artifacts` |
| Events (shared infrastructure) | `events`, `cfx_outbox` |

The AI context has **no owned tables**. AI operations are read-only from the AI package's perspective — they read job/note data and return generated text. The calling service persists any AI-generated content.

### 5. Package and service API surface rules

Every shared package (`@clark/*`) exports only what callers need to use:
- No internal implementation details (raw `query()`, `getAnthropicClient()`, crypto primitives)
- Exported interfaces are stable contracts — changes require versioning discussion
- Packages do not import from each other's internal modules (only from `index.ts` exports)

The `@clark/db` package is a special case: it provides the shared DB connection infrastructure. Repositories inside each service use it, but raw `query()` calls should not appear in route handlers or service methods — they belong in repository/query classes.

### 6. Event-driven coordination between contexts

Contexts do not call each other's service methods directly. Cross-context coordination happens via:
1. **Domain events** — context A appends an event; context B has an event handler that reacts
2. **Outbox messages** — context A writes a message to the outbox; external systems consume it

For Phase 1, domain event handlers run in-process (same Fastify process). In Phase 2, handlers can be moved to separate processes by switching the event bus from in-memory to a AMQP topic subscriber.

### 7. Testing principle

Every domain service class must be testable without starting the HTTP server or the database. Service classes receive their dependencies (repositories, event store, outbox writer) via constructor injection, enabling unit testing with test doubles.

---

## Consequences

**Easier:**
- New bounded contexts (Inspection, Firmware, AI agents) follow the same pattern from the start
- Service classes are unit-testable without HTTP or database
- External integrations can fail without blocking HTTP responses or domain state changes
- The codebase can be incrementally refactored toward separate deployable services

**Harder:**
- More files and more ceremony than the current transaction script pattern
- The `withTransaction` requirement for outbox writes adds complexity to service methods
- Developers must understand the layered pattern before contributing

**Breaking changes from current code:**
- `packages/ai/src/review-manager.ts` — remove direct `query()` call; AI functions return data, callers persist it
- `apps/api/src/routes/v1/jobs.ts` — remove inline SQL and CFX publish; delegate to `JobService`
- `apps/api/src/plugins/auth.ts` — remove direct `queryOne()`; delegate to `IdentityService`
- All route handlers — thin adapters only; no SQL, no business logic

**Migration priority (to be done before adding new contexts):**
1. Fix AI package data ownership violation (remove direct DB writes)
2. Fix CFX to use `cfx_outbox` table properly
3. Create `JobService` as the pattern reference
4. Apply pattern to `InspectionService` (Step 3) from the start

**Not doing yet:**
- Repository classes to abstract raw SQL (deferred to Phase 2 — too much ceremony for Phase 1 velocity)
- Separate packages per bounded context (deferred — single `apps/api` is fine for Phase 1)
- Event bus migration from in-process to AMQP (deferred to Phase 2)
