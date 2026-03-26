# Clarkware Agent Roster

This directory defines the specialized agents used in the Clarkware multi-agent development system. Each agent is a defined persona and scope that can be invoked in a Claude Code session to bring focused expertise to a specific domain of the platform.

---

## Agent Roster

| Agent | File | Domain | When to Invoke |
|-------|------|--------|----------------|
| Orchestrator | `orchestrator.md` | Cross-cutting planning, roadmap, ADRs | Starting a session, planning a sprint, resolving conflicts between domains |
| Firmware Engineer | `firmware-engineer.md` | GDB/DAP/OpenOCD/JTAG/PlatformIO/SVD/DWARF | Anything in the Firmware Panel or DeviceDriverManager |
| CFX Integration Engineer | `cfx-integration.md` | AMQP/RabbitMQ/IPC-CFX/CFXPublisher | Message schema design, broker topology, event mapping |
| Quality Inspector | `quality-inspector.md` | IPC-A-610, inspection workflow, workmanship criteria | Workmanship Panel, defect classification, compliance documentation |
| AI Platform Engineer | `ai-platform.md` | Theia AI, MCP, LLM communication, prompt design | AI agent wiring, MCP server integration, Theia AI contributions |

---

## How to Invoke an Agent

At the start of a Claude Code session focused on a specific domain, paste the agent's `## System Prompt` block as context. The agent definition scopes Claude's attention, vocabulary, and decision-making priorities to the relevant domain.

---

## Agent Interaction Pattern

For multi-domain tasks, use the Orchestrator agent first to scope the work, then switch to the domain-specific agent for implementation. Hand-offs are documented in `planning/progress-assessment.md`.
