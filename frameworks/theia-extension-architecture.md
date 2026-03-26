# Framework: Theia Extension Architecture

**Covers:** How Theia extensions are structured, InversifyJS DI, BackendApplicationContribution, WidgetFactory, JSON-RPC channel, frontend/backend service split, and how the existing Clark extensions should be refactored toward the target IPE architecture.

---

## 1. Theia's Core Concepts

### Extension vs Plugin

| Type | Mechanism | Packaging | When to use |
|------|-----------|-----------|-------------|
| **Extension** | InversifyJS DI contributions | Built into the application at compile time | All Clark IPE features — full access to Theia internals |
| **Plugin** | VS Code extension API | Loaded at runtime (can be added/removed) | Third-party VS Code extensions: `cortex-debug`, `platformio-ide`, clangd |

Clark builds extensions (compiled in). VS Code ecosystem tools are loaded as plugins via Theia's plugin support.

### InversifyJS Dependency Injection

Theia uses InversifyJS for DI throughout. Every service, widget, and contribution is registered in a `ContainerModule` and injected via `@inject()`:

```typescript
// In clark-core-extension/src/browser/clark-frontend-module.ts
export default new ContainerModule(bind => {
  bind(JobService).toSelf().inSingletonScope();
  bind(FrontendApplicationContribution).toService(JobService);
  bind(WidgetFactory).toDynamicValue(ctx => ({
    id: CLARK_WIDGET_ID,
    createWidget: () => ctx.container.get<ClarkWidget>(ClarkWidget)
  }));
});
```

---

## 2. Frontend / Backend Split

Every Theia extension has two sides:

| Side | Runtime | Location | Purpose |
|------|---------|----------|---------|
| **Frontend** | Browser / Electron renderer | `src/browser/` | UI widgets, commands, menus, keybindings |
| **Backend** | Node.js | `src/node/` | File system, hardware, network, DB, process spawning |

They communicate via **JSON-RPC over WebSocket** using Theia's `IPC` mechanism:

```typescript
// backend (node) — exposes a service
export const JOB_SERVICE_PATH = '/services/clark-job';

@injectable()
export class JobServiceImpl implements JobService {
  async getJob(id: string): Promise<Job> { /* DB call */ }
}

// frontend (browser) — consumes via JSON-RPC proxy
@injectable()
export class JobServiceClient {
  @inject(WebSocketConnectionProvider)
  protected wsProvider: WebSocketConnectionProvider;

  protected proxy: JobService;

  @postConstruct()
  init(): void {
    this.proxy = this.wsProvider.createProxy<JobService>(JOB_SERVICE_PATH);
  }

  getJob(id: string): Promise<Job> {
    return this.proxy.getJob(id);
  }
}
```

---

## 3. Key Contribution Points

### BackendApplicationContribution

A service that participates in the backend application lifecycle:
```typescript
@injectable()
export class CFXPublisher implements BackendApplicationContribution {
  async initialize(): Promise<void> {
    await this.connectToBroker(); // called once on backend startup
  }
  onStop(): void {
    this.disconnect();
  }
}
```

Use this for: CFXPublisher, DeviceDriverManager, MCP server lifecycle, AMQP connection.

### FrontendApplicationContribution

Same pattern for the browser side:
```typescript
@injectable()
export class ClarkLayoutContribution implements FrontendApplicationContribution {
  async initializeLayout(app: FrontendApplication): Promise<void> {
    await this.widgetManager.getOrCreateWidget(CLARK_WIDGET_ID);
    // position widgets in the shell
  }
}
```

**Note:** The existing prototype has a manual patch in `src-gen/frontend/index.js` to work around `initializeLayout` not firing. The target architecture should use `FrontendApplicationContribution.initializeLayout` properly — this eliminates the manual patch risk.

### WidgetFactory

Creates and manages Theia widget instances:
```typescript
bind(WidgetFactory).toDynamicValue(ctx => ({
  id: FIRMWARE_PANEL_WIDGET_ID,
  createWidget: () => {
    const child = ctx.container.createChild();
    child.bind(FirmwareWidget).toSelf();
    return child.get(FirmwareWidget);
  }
})).inSingletonScope();
```

### StatusBarContribution

For the Network Status Bar:
```typescript
@injectable()
export class NetworkStatusBarContribution implements FrontendApplicationContribution {
  @inject(StatusBar) protected statusBar: StatusBar;
  @inject(ICFXPublisher) protected cfxPublisher: ICFXPublisher;

  onStart(): void {
    this.cfxPublisher.onConnectionChange(status => {
      this.statusBar.setElement('clark-cfx-status', {
        text: status === 'connected' ? '$(radio-tower) CFX' : '$(warning) CFX Offline',
        tooltip: `CFX Broker: ${status}`,
        alignment: StatusBarAlignment.RIGHT,
        priority: 100
      });
    });
  }
}
```

---

## 4. Current Extension Structure and Target Refactor

### Current (P1V1 prototype)

```
apps/ipe/src/extensions/
├── clark-core-extension/          — job list, job context, notes, auto-login
└── clark-messaging-extension/     — WebSocket event stream (not real XMPP)
```

### Target Extension Structure

```
apps/ipe/src/extensions/
├── clark-core-extension/          — keep: job, workstation, notes, presence
├── clark-cfx-extension/           — NEW: CFXPublisher, AMQP, NetworkStatusBar
├── clark-firmware-extension/      — NEW: DeviceDriverManager, DAP config, SVD, PlatformIO
├── clark-inspection-extension/    — NEW: Workmanship Panel, IPC criteria, defect log
├── clark-ai-extension/            — NEW: Theia AI agents, MCP tool provider, audit trail
└── clark-messaging-extension/     — refactor: real XMPP chat (vs current WS event stream)
```

Each extension has this structure:
```
clark-<domain>-extension/
├── package.json                   — extension metadata, theiaExtensions declaration
├── tsconfig.json
├── src/
│   ├── browser/                   — frontend contributions (widgets, commands)
│   │   └── clark-<domain>-frontend-module.ts
│   └── node/                      — backend services
│       └── clark-<domain>-backend-module.ts
└── webpack.config.js              — only if the extension has its own bundle entry
```

---

## 5. Key Known Issues to Fix in Prototype

| Issue | Location | Fix |
|-------|----------|-----|
| `src-gen/frontend/index.js` manual patch | `apps/ipe/src-gen/frontend/index.js` | Implement `FrontendApplicationContribution.initializeLayout` in `clark-core-extension` |
| Auto-login hardcoded | `clark-widget.tsx` `doLogin()` | Move to a proper auth contribution with session management |
| Compiled bundle committed to git | `apps/ipe/lib/frontend/bundle.js` | Add to `.gitignore`; document the build step |
| Messages panel shows WebSocket events not XMPP | `clark-messaging-extension` | Wire real XMPP client from `@clark/messaging` |

---

## 6. Theia Version Considerations

Current: **Theia 1.69**

Theia AI features were introduced in **1.54+** (2024). The current 1.69 version includes Theia AI but the Anthropic provider may require a custom contribution rather than a built-in one.

**Recommendation:** Plan to upgrade to **Theia 1.75+** (latest stable as of Q1 2026) to get the most mature Theia AI tooling. Verify VS Code plugin API compatibility before upgrading — the `platformio-ide` and `cortex-debug` extensions require Theia's plugin API to be at a compatible version.
