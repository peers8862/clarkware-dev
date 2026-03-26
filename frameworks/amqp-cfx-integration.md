# Framework: AMQP Broker and IPC-CFX Integration

**Covers:** AMQP protocol choice, RabbitMQ deployment, CFXPublisher Theia service design, tiered broker topology, offline buffering, schema validation, and the `@clark/cfx-schemas` package.

---

## 1. Protocol Choice: AMQP 0.9.1 vs AMQP 1.0

The IPC-CFX specification (IPC-2591) references AMQP 1.0 (ISO/IEC 19464) as its transport. In practice:

| Factor | AMQP 1.0 | AMQP 0.9.1 (RabbitMQ native) |
|--------|----------|------------------------------|
| CFX spec compliance | Formal compliance | Informal — widely accepted in industry |
| Node.js ecosystem | `rhea`, `rhea-promise` | `amqplib` (mature, well-maintained) |
| RabbitMQ support | Plugin (`rabbitmq_amqp1_0`) — less tested | Native — fully supported |
| Feature parity for Clark's needs | No missing features for Clark's use case | No missing features |
| Recommendation | Use for formal CFX certification | **Preferred for Clark Phase 1** |

**Decision for Phase 1:** Use AMQP 0.9.1 with `amqplib` and RabbitMQ. This is the path of least resistance, the most mature Node.js tooling, and adequate for Clark's broker topology. Enable the `rabbitmq_amqp1_0` plugin for future formal certification testing.

---

## 2. RabbitMQ Deployment

### Local workstation (development and single-node production)

```yaml
# docker/compose.yml addition
rabbitmq:
  image: rabbitmq:3.13-management-alpine
  container_name: clark-broker
  ports:
    - "5672:5672"     # AMQP
    - "15672:15672"   # Management UI (dev only)
  environment:
    RABBITMQ_DEFAULT_USER: clark
    RABBITMQ_DEFAULT_PASS: clark_dev
  volumes:
    - rabbitmq_data:/var/lib/rabbitmq
  healthcheck:
    test: rabbitmq-diagnostics ping
    interval: 10s
    timeout: 5s
    retries: 5
```

### Exchange and queue topology

```
Exchange: clark.cfx (type: topic, durable: true)
  ├── Binding: CFX.#              → queue: clark.cfx.all       (full audit)
  ├── Binding: CFX.WorkOrder*     → queue: clark.cfx.workorder  (job tracking)
  ├── Binding: CFX.Inspection*    → queue: clark.cfx.inspection (quality)
  ├── Binding: com.clark.ipe.*    → queue: clark.cfx.custom     (Clark extensions)
  └── Dead letter exchange: clark.cfx.dlx → queue: clark.cfx.dead
```

### Routing key convention

`<namespace>.<MessageName>.<NodeId>.<WorkstationId>`

Examples:
- `CFX.WorkOrderStarted.node-niagara-01.ws-smt-03`
- `CFX.InspectionCompleted.node-niagara-01.ws-smt-03`
- `com.clark.ipe.FirmwareProvisioned.node-niagara-01.ws-fw-02`

---

## 3. CFXPublisher — Theia Backend Service Design

### Location

`apps/ipe/src/extensions/clark-cfx-extension/src/node/cfx-publisher.ts`

### Interface

```typescript
export const ICFXPublisher = Symbol('ICFXPublisher');

export interface ICFXPublisher {
  /** Publish a CFX message. Returns immediately — delivery is guaranteed via local queue. */
  publish(messageName: string, payload: object): void;

  /** Connection status observable — emits on connect/disconnect/reconnect */
  readonly onConnectionChange: Event<CFXConnectionStatus>;

  /** Current connection status */
  readonly connectionStatus: CFXConnectionStatus;
}

export type CFXConnectionStatus = 'connected' | 'disconnected' | 'reconnecting';
```

### Implementation responsibilities

1. **Connection management** — Connect on `BackendApplicationContribution.initialize()`. Auto-reconnect with exponential backoff (100ms → 200ms → 400ms → ... capped at 30s).
2. **Message assembly** — Wrap each payload in the CFX envelope: `MessageName`, `Version`, `TimeStamp`, `UniqueIdentifier`, `Source` (CFX Handle).
3. **Local buffer queue** — On publish, write to a PostgreSQL `cfx_outbox` table with status `pending`. A background flush loop reads `pending` rows and publishes to RabbitMQ. On success, mark `delivered`. On repeated failure, mark `dead`.
4. **Schema validation** — Validate payload against `@clark/cfx-schemas` before queueing. Log and dead-letter invalid messages.
5. **Status bar event emission** — Emit `CFXConnectionStatus` changes on the Theia `IEventBus` so the Network Status Bar widget can display broker connection status.

### CFX outbox table (PostgreSQL)

```sql
CREATE TABLE cfx_outbox (
  id              BIGSERIAL PRIMARY KEY,
  message_name    TEXT NOT NULL,
  routing_key     TEXT NOT NULL,
  payload         JSONB NOT NULL,
  status          TEXT NOT NULL DEFAULT 'pending',   -- pending | delivered | dead
  attempts        INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  delivered_at    TIMESTAMPTZ,
  error           TEXT
);
CREATE INDEX ON cfx_outbox(status, created_at) WHERE status = 'pending';
```

---

## 4. Offline Mode and Tiered Topology

### Tier 1: Local workstation (always-on)

- RabbitMQ runs on the workstation's Docker host
- CFXPublisher connects to `amqp://localhost:5672`
- Fully functional without any network connectivity
- Outbox queue provides durability across IPE restarts

### Tier 2: Node-level federation (facility network)

- One RabbitMQ instance per facility node
- Workstation brokers federate upward using the **Shovel plugin**
- Shovel configuration: all `clark.cfx` messages shovelled to `clark.cfx` exchange on node broker
- Node broker aggregates events from all workstations

### Tier 3: Clark platform (internet/WAN)

- Node brokers connect to Clark's cloud AMQP broker (same Shovel pattern)
- TLS required at this tier
- Clark platform consumes events for reporting, customer portal, cross-node orchestration

---

## 5. @clark/cfx-schemas Package

### Purpose

TypeScript type definitions and JSON Schema validators for all CFX message types used by the IPE.

### Structure

```
packages/cfx-schemas/
├── src/
│   ├── types/
│   │   ├── WorkOrderStarted.ts
│   │   ├── InspectionCompleted.ts
│   │   ├── NonConformanceCreated.ts
│   │   ├── OperatorActivity.ts         # CFX 2.0
│   │   ├── WorkInstructionCompleted.ts # CFX 2.0
│   │   ├── SignoffRequired.ts          # CFX 2.0
│   │   ├── SignoffCompleted.ts         # CFX 2.0
│   │   └── clark/
│   │       └── FirmwareProvisioned.ts  # Clark extension
│   ├── validate.ts                     # AJV-based validator
│   └── index.ts
└── schemas/                            # Raw JSON Schema files (source of truth)
    ├── WorkOrderStarted.json
    └── ...
```

### Usage

```typescript
import { validateCFXMessage, WorkOrderStartedPayload } from '@clark/cfx-schemas';

const payload: WorkOrderStartedPayload = { ... };
const result = validateCFXMessage('CFX.WorkOrderStarted', payload);
if (!result.valid) {
  logger.error('Invalid CFX message', result.errors);
}
```

---

## 6. Outstanding Questions / Decisions Required

- [ ] RabbitMQ version: 3.13 (current stable) — confirmed
- [ ] Should the outbox queue use PostgreSQL (shared with main DB) or a separate SQLite file (simpler, no DB dependency at startup)?
- [ ] Shovel vs Federation for tier 2→3 routing? Shovel is simpler; Federation is more flexible for bidirectional message flows.
- [ ] Should Clark join the IPC CFX Working Group for access to early 2.1 drafts?
- [ ] CFX Handle format — confirm `clark.ipe.<node-id>.<workstation-id>` convention with operations team
