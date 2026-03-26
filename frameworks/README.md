# Frameworks

Deep technical integration specifications for the Clarkware IPE platform. Each document covers a specific technical domain in implementation-level detail — not conceptual overview (that's in `docs/`) but the specific interfaces, packages, patterns, and decisions needed to build it.

---

## Index

| Document | Domain | Key Topics |
|----------|--------|-----------|
| [amqp-cfx-integration.md](amqp-cfx-integration.md) | AMQP / IPC-CFX | RabbitMQ, CFXPublisher, cfx_outbox, tiered topology, @clark/cfx-schemas |
| [firmware-toolchain-integration.md](firmware-toolchain-integration.md) | Firmware / Embedded | GDB, DAP, OpenOCD, J-Link, JTAG/SWD, PlatformIO, clangd, SVD, ELF/DWARF, RTOS |
| [ai-llm-mcp-integration.md](ai-llm-mcp-integration.md) | AI / LLM / MCP | Theia AI, MCP server, Claude/Ollama providers, AI audit trail, tool definitions |
| [theia-extension-architecture.md](theia-extension-architecture.md) | Theia / Extension System | InversifyJS DI, BackendApplicationContribution, WidgetFactory, JSON-RPC, extension decomposition |
