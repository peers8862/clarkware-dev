# ADR-001: AMQP Broker Topology and CFX Transport

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

IPC-CFX (IPC-2591) specifies AMQP as its transport. The IPE must publish CFX messages for every significant production event. The system must work in fully offline environments (no network connectivity), in facility-local networks, and when connected to Clark's cloud platform.

Two AMQP protocol versions exist: AMQP 0.9.1 (RabbitMQ native) and AMQP 1.0 (ISO standard, referenced by CFX spec). Two broker deployment models are possible: a single central broker or a tiered hierarchy.

---

## Decision

**1. Protocol: AMQP 0.9.1 with RabbitMQ.** Use `amqplib` as the Node.js client. Enable the `rabbitmq_amqp1_0` plugin for future formal CFX certification testing but do not require AMQP 1.0 compliance for Phase 1.

**2. Tiered topology:**
- Tier 1 (workstation): RabbitMQ on the local Docker host — always-on, no network dependency
- Tier 2 (node/facility): One RabbitMQ instance per facility — workstations shovel messages upward
- Tier 3 (Clark platform): Clark cloud AMQP broker — facility brokers shovel upward over TLS

**3. Offline durability:** A `cfx_outbox` table in PostgreSQL buffers messages when the broker is unreachable. A background flush loop replays them. This means the Clark domain event store and CFX outbox share the same PostgreSQL instance — one infrastructure dependency, not two.

**4. CFXPublisher is a BackendApplicationContribution** — runs in the Theia Node.js backend, never in the browser.

---

## Consequences

**Easier:**
- AMQP 0.9.1 + `amqplib` is a mature, well-documented path with abundant Node.js examples
- RabbitMQ's management UI enables easy debugging of message flow during development
- Tiered topology supports both air-gapped nodes and connected nodes without code changes

**Harder:**
- Formal CFX certification testing requires an AMQP 1.0 endpoint — needs the `rabbitmq_amqp1_0` plugin
- The `cfx_outbox` table adds a migration and a background job to manage
- Shovel plugin configuration adds operational complexity at the node level (but this is a deployment concern, not a code concern)

**Superseded by:** Nothing yet.
