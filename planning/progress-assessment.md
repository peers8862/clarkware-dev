# Clarkware — Progress Assessment and Next Steps

**Last updated:** 2026-03-26 (session 3)

---

## Current State Summary

### What exists in `clarkware/` (P1V1 prototype)

The prototype is a working, running system. The following are fully functional:

| Feature | Status |
|---------|--------|
| Theia IPE shell (browser-based) | Working |
| Job list, create, lifecycle (start/pause/resume/complete/void/reopen) | Working |
| Job context panel (live detail view) | Working |
| Notes — view and post, append-only | Working |
| Real-time WebSocket event stream per job | Working |
| Auth — login, token refresh, logout, JWT | Working |
| RBAC enforcement on all protected routes | Working |
| AI routes (5 endpoints via Anthropic SDK) | Working (requires API key) |
| Artifact upload/download (MinIO presigned URLs) | Working |
| PostgreSQL domain event store | Working |
| Docker Compose infra (PG, MinIO, Prosody, OpenSearch) | Working |

| Feature | Status |
|---------|--------|
| Issues UI | Scaffolded (no UI) |
| Conversations (XMPP) | Backend complete, UI shows WS events not real XMPP |
| Shifts (clock-in/out) | Scaffolded (no UI) |
| Presence | Scaffolded (no UI) |
| Reporting dashboard | Read-model queries exist, no UI |
| Offline sync | Conflict handler implemented, not activated |

### Known technical debt in prototype

| Issue | Risk Level |
|-------|-----------|
| ~~`src-gen/frontend/index.js` manual patch~~ **FIXED** — panel layout moved to `ClarkFrontendContribution.onStart()`, `src-gen/` gitignored | ~~HIGH~~ Done |
| Auto-login hardcoded (admin/admin_dev_password) | MEDIUM |
| Compiled bundle committed to git | LOW |
| Messages panel shows WS events not real XMPP | MEDIUM |

### What exists in `clarkware-dev/`

- `docs/` — 14 strategy/architecture documents (fully written)
- `agents/` — 5 agent definitions (Orchestrator, Firmware Engineer, CFX Integration, AI Platform, Quality Inspector)
- `frameworks/` — 4 technical integration specifications (AMQP/CFX, Firmware toolchain, AI/LLM/MCP, Theia extension architecture)
- `decisions/` — 6 ADRs (AMQP topology, GDB/DAP, MCP/Theia AI, PlatformIO/RTOS, SVD/ELF, Extension structure)
- `planning/` — this document

---

## Gap Analysis: Prototype → Target IPE Architecture

### Layer 1: What exists but needs refactoring

| Gap | Effort | Priority |
|-----|--------|---------|
| `FrontendApplicationContribution.initializeLayout` — replace manual patch | Medium | P1 |
| Real XMPP integration in Messages panel (use `@clark/messaging`) | Medium | P2 |
| Proper login/auth UI (replace hardcoded auto-login) | Medium | P1 |

### Layer 2: What's entirely new (P1 scope)

| New Component | Complexity | ADR |
|--------------|-----------|-----|
| `clark-cfx-extension` + CFXPublisher + RabbitMQ Docker service | Medium | ADR-001 |
| `clark-firmware-extension` + DeviceDriverManager + OpenOCD/JLink | High | ADR-002, ADR-004, ADR-005 |
| `clark-inspection-extension` + IPC criteria DB + Workmanship Panel | High | None yet |
| `clark-ai-extension` + Theia AI agents + MCP server | Medium | ADR-003 |
| `cfx_outbox` PostgreSQL table + background flush loop | Low | ADR-001 |
| `@clark/cfx-schemas` TypeScript package | Low | ADR-001 |
| `packages/mcp-server` | Medium | ADR-003 |
| `packages/firmware-utils` (ELF hash, version extract, SVD library) | Low | ADR-005 |

### Layer 3: What's new (P2 scope — defer)

| New Component | Complexity |
|--------------|-----------|
| Reporting dashboard | Medium |
| Shift/presence UI | Low |
| Issues UI | Low |
| Tiered AMQP broker topology (Shovel to node + cloud) | Medium |
| RTOS Zephyr awareness | Low |
| MCP SSE transport (external agent access) | Low |
| Cortex-Debug RTOS awareness for non-FreeRTOS | Low |
| Clark Firmware SDK (standard ELF version symbol) | Medium |

---

## Next Actions (Recommended Phase 1 sequence)

### ~~Step 1 — Fix `initializeLayout`~~ DONE
- `ClarkFrontendContribution.onStart()` now opens all four panels (main, job-context, notes, messaging)
- `initializeLayout` kept as a fallback for fresh layouts
- `MESSAGING_WIDGET_ID` referenced by string constant to avoid hard import dep on messaging extension
- `apps/ipe/src-gen/` and `lib/frontend/bundle.js*` added to `.gitignore`
- Manual patch in `src-gen/frontend/index.js` removed

### ~~Step 2 — Add RabbitMQ to Docker Compose and implement CFXPublisher~~ DONE
- RabbitMQ 3.13-management added to `docker/compose.yml` (AMQP :5672, mgmt UI :15672)
- `rabbitmq_data` volume added
- `cfx_outbox` table added to `docker/postgres/init.sql`
- `packages/cfx/` — new `@clark/cfx` package with `CFXClient` and `CFX_MESSAGES`
- `clark-cfx-extension` — new Theia extension with `CFXPublisher` (BackendApplicationContribution) and `CFXStatusBarContribution` (shows broker status in status bar)
- `apps/api/src/plugins/cfx.ts` — Fastify plugin that mounts `CFXClient` as `fastify.cfx`
- `apps/api/src/routes/v1/jobs.ts` — wired: `WorkOrderStarted` on job start, `WorkOrderCompleted` on job complete
- `apps/ipe/package.json` — `clark-cfx-extension` added as workspace dep
- `apps/api/package.json` and `tsconfig.json` — `@clark/cfx` added

**To activate:** Run `pnpm install` then `pnpm build` to compile the new `@clark/cfx` package and `clark-cfx-extension`.

### ~~Step 3 — IPC Criteria Database and Inspection Service~~ DONE
- `packages/ipc-criteria/` — `@clark/ipc-criteria` package with 8 IPC-A-610 workmanship criteria (Class 1/2/3), type definitions, `getCriteria()`, `getCriterionById()`, `searchCriteria()`
- `apps/api/src/services/inspection-service.ts` — full `InspectionService` (ADR-007 compliant): `createStep`, `logInspectionPoint` (atomic: result + defect + domain event + CFX outbox), `completeStep` (atomic: update + domain event + `INSPECTION_COMPLETED` CFX), `dispositionDefect`, all queries
- `apps/api/src/routes/v1/inspection.ts` — thin adapter, 9 endpoints, no SQL or business logic
- `docker/postgres/init.sql` — Section 17: `inspection_steps`, `inspection_results`, `defect_records` tables
- `apps/api/package.json` + `tsconfig.json` — `@clark/ipc-criteria` dependency added
- `apps/api/src/app.ts` — inspection routes registered
- **Service compliance tracker** added: `clarkware-dev/service-compliance/` — one `.md` per bounded context documenting compliant vs. non-compliant route files

**Remaining for Step 3 — Theia UI (deferred to after Step 4):**
- `clark-inspection-extension` Theia extension — Workmanship Panel widget displaying criteria for current job

### ~~Step 4 — Firmware Panel (flash-only mode)~~ DONE
- `packages/firmware-utils/` — `@clark/firmware-utils` package: `elfHash` (SHA-256 via Node crypto streams), `elfVersion` (arm-none-eabi-readelf subprocess), `probeList` (VID/PID table for J-Link, ST-Link, CMSIS-DAP, Black Magic)
- `apps/api/src/services/firmware-service.ts` — `FirmwareService` (ADR-007 compliant): `createRecord` (pre-flash, records binary hash), `recordFlashResult` (atomic: DB update + domain event + `FIRMWARE_PROVISIONED` CFX outbox on success), `getRecordsForJob`
- `apps/api/src/routes/v1/firmware.ts` — thin adapter, 3 endpoints, no SQL or business logic
- `docker/postgres/init.sql` — Section 18: `firmware_records` table
- `apps/api/package.json` + `tsconfig.json` — `@clark/firmware-utils` dependency added
- `apps/api/src/app.ts` — firmware routes registered
- `clark-firmware-extension` — Theia extension: `FirmwareWidget` (React, listens for `clark:job-selected`, shows records per job, new flash form with client-side SHA-256 via SubtleCrypto, result recording form), `firmware-api.ts` client, `frontend-module.ts`
- `apps/ipe/package.json` — `clark-firmware-extension` added as workspace dep
- `clark-core-extension/src/frontend-module.ts` — `FIRMWARE_WIDGET_ID` added, panel opened at `bottom` rank 400

**Phase 1 scope note:** Flash-only mode. No probe enumeration, no OpenOCD/J-Link process spawning, no DAP/debug session. Operator flashes with existing tooling; IPE captures the provenance record. Full automated flash pipeline is Phase 2.

### Step 5 — Theia AI InspectionAssistant
- Create `clark-ai-extension` with `InspectionAssistant` agent
- Wire to `@clark/ai` Anthropic SDK (already exists)
- Integrate with `clark-inspection-extension` — operator can ask AI to classify a defect
- Record all AI responses in `ai_audit_records` table
- Display AI outputs with explicit "AI-generated" label in Audit Panel
- **Why now:** Completes the AI loop needed for milestone M1.4.

---

## Milestone Tracker (from roadmap)

| Milestone | Status | Blocker |
|-----------|--------|---------|
| M1.1: IPE shell operational at pilot station | In progress — prototype works in dev | Needs `initializeLayout` fix and proper auth |
| M1.2: XMPP messaging operational for pilot users | Not started | `@clark/messaging` backend complete, UI not wired |
| M1.3: First evidence import from pilot tool | Not started | Firmware Panel not built |
| M1.4: First AI-assisted summary or routing loop | In progress — AI routes exist but not Theia-integrated | Theia AI extension not built |
| M1.5: First exportable record package | Not started | Audit Panel not built |

---

## Open Architectural Questions

1. **Theia version upgrade:** Current is 1.69. Theia AI is most mature in 1.75+. Plan an upgrade before building the AI extension.
2. **J-Link license procurement:** J-Link BASE ~$400/unit. Confirm with CLARK ops team before finalizing the probe strategy.
3. **Air-gapped LLM model:** `mistral-7b` or `phi3:medium` for Ollama deployment? Depends on available GPU hardware at nodes.
4. **IPC criteria database licensing:** IPC-A-610 is a paid standard. The criteria database must be authored from scratch (structured summaries, not verbatim text) to avoid licensing issues.
5. **MCP server transport:** stdio (local) vs SSE (HTTP remote). SSE enables external agents. Recommendation: stdio for Phase 1.
