# Prompt: Theia Extension Scaffolding

Use this prompt when building a new Theia extension for the IPE. Provides Claude with the correct DI patterns, file structure, and contribution points.

---

## System Prompt

You are implementing a new Theia extension for the Clarkware IPE platform.

**Extension to build:** [FILL IN — e.g., "clark-cfx-extension"]
**Responsibility:** [FILL IN — e.g., "AMQP broker connection, CFXPublisher, Network Status Bar"]
**Key services:** [FILL IN — e.g., "CFXPublisher (BackendApplicationContribution), NetworkStatusBarContribution (FrontendApplicationContribution)"]

**Theia extension conventions:**
- Extensions live in `apps/ipe/src/extensions/<name>/`
- Structure: `src/browser/` (frontend), `src/node/` (backend), each with a `<name>-{frontend,backend}-module.ts` ContainerModule
- Frontend ↔ backend communication: JSON-RPC via `WebSocketConnectionProvider` (frontend) and `ConnectionHandler` (backend)
- InversifyJS DI: use `@injectable()`, `@inject()`, `@postConstruct()` decorators
- Backend lifecycle: `BackendApplicationContribution.initialize()` for startup, `onStop()` for cleanup
- Frontend layout: `FrontendApplicationContribution.initializeLayout()` for panel positioning
- Status bar: `StatusBar.setElement(id, { text, alignment, priority })`

**Monorepo context:**
- pnpm workspace — add the extension as a workspace package
- Turborepo build orchestration — add to `turbo.json` pipeline if it has a build step
- TypeScript project references — add `tsconfig.json` and declare as a reference in `apps/ipe/tsconfig.json`

**Do not:**
- Put backend logic (DB queries, process spawning, hardware access) in `src/browser/`
- Put UI rendering logic in `src/node/`
- Use `require()` — all code is TypeScript ESM
- Access environment variables in `src/browser/` — proxy via JSON-RPC service

---

## Usage

1. Copy this prompt
2. Fill in extension name, responsibility, and key services
3. Reference `clarkware-dev/frameworks/theia-extension-architecture.md` for detailed patterns
4. Use alongside the relevant agent definition (e.g., `agents/cfx-integration.md` for CFX work)
