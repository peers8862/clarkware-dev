<<<<<<< HEAD
# clarkware-dev
=======
# clarkware-dev

**Development Intelligence Repository for the Clarkware IPE Platform**

This folder is the planning, memory, and architectural intelligence layer for the Clarkware project. It does not contain production code — that lives in `../clarkware/`. Instead it holds the structured knowledge, decision records, agent definitions, technical frameworks, and iterative planning artefacts that guide how the production code evolves.

---

## Folder Structure

```
clarkware-dev/
├── README.md                          — this file
├── docs/                              — conceptual and strategic documents (existing)
├── agents/                            — multi-agent definitions: personas, tools, scopes
├── frameworks/                        — deep technical integration specifications
├── decisions/                         — Architecture Decision Records (ADRs)
├── prompts/                           — reusable prompts for planning and scaffolding sessions
├── workflows/                         — step-by-step operational and development workflows
├── planning/                          — current state assessments and sprint-level planning
└── service-compliance/                — per-context tracker: which API routes are thin adapters vs. still have SQL/logic
```

---

## Navigation

| Need | Go to |
|------|-------|
| Understand the project concept and business model | `docs/clarkware.md` |
| Understand the IPE architecture (Theia + CFX) | `docs/clark_ipe_architecture.md` |
| Understand the data model | `docs/clarkware-data-model.md` |
| Understand the messaging model | `docs/clarkware-messaging-model.md` |
| Understand the roadmap | `docs/clarkware-roadmap.md` |
| Understand the AMQP/CFX integration stack | `frameworks/amqp-cfx-integration.md` |
| Understand the firmware toolchain (GDB/DAP/OpenOCD/JTAG/SVD/ELF) | `frameworks/firmware-toolchain-integration.md` |
| Understand Theia AI / MCP / LLM integration | `frameworks/ai-llm-mcp-integration.md` |
| Understand how Theia extensions are structured | `frameworks/theia-extension-architecture.md` |
| See all architectural decisions and their rationale | `decisions/` |
| Run a planning or architecture session | `prompts/` |
| Understand the current build state and next steps | `planning/progress-assessment.md` |
| Check which API route files are compliant with ADR-007 (no SQL in routes) | `service-compliance/README.md` |

---

## Multi-Agent System

This repo supports a multi-agent development model. Different agents specialize in different domains of the platform. See `agents/README.md` for the full agent roster and how to invoke each agent in a planning or implementation session.

---

## How to Use This Repo in a Claude Code Session

1. Start a session by reading `planning/progress-assessment.md` for current state.
2. Use the relevant `agents/` definition to orient Claude on the domain being worked on.
3. Reference the relevant `frameworks/` document for the specific technical area.
4. Check `decisions/` before making any architectural choice — the ADR may already be resolved.
5. Add new ADRs for any significant architectural decision made during the session.
6. Update `planning/progress-assessment.md` at the end of each session.
>>>>>>> 6a0d611 (large initial commit for development repo for clarkware)
