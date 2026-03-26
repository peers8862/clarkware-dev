# ADR-006: Theia Extension Decomposition into Domain Extensions

**Status:** Proposed
**Date:** 2026-03-26

---

## Context

The current prototype has two Theia extensions: `clark-core-extension` (jobs, notes, messaging) and `clark-messaging-extension` (WebSocket event stream). As the platform grows to include CFX/AMQP, firmware toolchain, workmanship inspection, and AI agents, continuing to grow `clark-core-extension` will create an unmaintainable monolith.

The question is: how should the IPE's Theia extensions be decomposed?

---

## Decision

Decompose into five domain extensions, each with a clear single responsibility:

| Extension | Responsibility | Priority |
|-----------|---------------|---------|
| `clark-core-extension` | Job management, notes, workstation context, auth session, `FrontendApplicationContribution` layout | P1 (exists, refactor) |
| `clark-cfx-extension` | CFXPublisher, AMQP connection, Network Status Bar, CFX outbox flush | P1 (new) |
| `clark-firmware-extension` | DeviceDriverManager, DAP config, OpenOCD/JLink process management, SVD manager, PlatformIO service, Firmware Panel widget | P1 (new) |
| `clark-inspection-extension` | Workmanship Panel, IPC criteria database, defect records, Non-conformance workflow | P1 (new) |
| `clark-ai-extension` | Theia AI agents, MCP ToolProvider bridge, AI audit trail, PromptTemplate registrations | P1 (new, but InspectionAssistant only in Phase 1) |
| `clark-messaging-extension` | Real XMPP chat (refactor from WebSocket events to true `@clark/messaging` integration) | P2 (refactor) |

### Extension dependency graph

```
clark-core-extension          ← no clark deps (base layer)
clark-cfx-extension           ← clark-core-extension (workstation context, job events)
clark-firmware-extension      ← clark-core-extension, clark-cfx-extension (emits FirmwareProvisioned)
clark-inspection-extension    ← clark-core-extension, clark-cfx-extension (emits InspectionCompleted)
clark-ai-extension            ← clark-core-extension, clark-inspection-extension
clark-messaging-extension     ← clark-core-extension
```

### Layout coordination

The `clark-core-extension`'s `FrontendApplicationContribution.initializeLayout` orchestrates the initial panel layout. Other extensions register their widgets and the layout contribution positions them. This eliminates the current manual patch in `src-gen/frontend/index.js`.

---

## Consequences

**Easier:**
- Each extension has a single owner and clear scope
- Extensions can be loaded or excluded per deployment type (e.g., a reporting-only node doesn't need `clark-firmware-extension`)
- Testing is scoped per extension
- The manual `src-gen/frontend/index.js` patch is eliminated

**Harder:**
- Five extensions require five `package.json` files, five `ContainerModule` registrations, and five TypeScript compilation steps
- Inter-extension dependencies must be managed via well-defined service interfaces — avoid circular dependencies
- The current `clark-core-extension` refactor is a non-trivial task — must be done carefully to avoid breaking the working prototype

**Migration path:**
1. Create the new extension shell structures (package.json, tsconfig.json, empty modules) — non-breaking
2. Implement `clark-cfx-extension` fresh — no existing code to migrate
3. Implement `clark-firmware-extension` fresh
4. Implement `clark-inspection-extension` fresh
5. Implement `clark-ai-extension` fresh
6. Gradually migrate `clark-core-extension` to use proper `FrontendApplicationContribution.initializeLayout`
7. Refactor `clark-messaging-extension` to real XMPP (Phase 2)
