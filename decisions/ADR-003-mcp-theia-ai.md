# ADR-003: MCP as the AI Tool Protocol and Theia AI as the Agent Framework

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

The IPE requires AI agents that can both advise (generate text) and act (create notes, log inspection results, query job history). This requires a tool-calling protocol. The existing prototype uses direct Anthropic SDK calls in `@clark/ai` — this is functional but not integrated with the Theia UI or extendable by third parties.

Options evaluated:
- Continue with direct SDK calls in the Fastify API backend (`@clark/ai`)
- Integrate Theia AI framework for in-IPE agents with native UI integration
- Build an MCP server to expose IPE tools to any MCP-compatible client
- Use an emerging protocol like OpenClaw or A2A

---

## Decision

**1. Theia AI as the in-IPE agent framework.**

Theia AI provides native integration points (ChatWidget, PromptTemplate, AIVariableService) that map directly to the IPE's needs. Agents that run inside the IPE use Theia AI's `AIAgent` contribution point. This gives operators a consistent, auditable UI for AI interactions within the IPE shell.

**2. MCP server (`packages/mcp-server`) for external and cross-tool AI access.**

The Clark IPE exposes an MCP server that makes IPE resources and tools accessible to any MCP-compatible client: Claude Desktop, external agents, customer integration tools. This enables AI use cases that live outside the IPE shell (e.g., a supervisor using Claude Desktop to query job records).

**3. Theia AI uses the MCP server via a `ToolProvider` wrapper.**

Rather than implementing tool logic twice (once in Theia AI contributions, once in the MCP server), Theia AI agents call the MCP server via a thin `MCPToolProvider` wrapper. One implementation, two access paths.

**4. Defer OpenClaw and A2A.** Neither protocol has sufficient maturity or tooling for Phase 1. MCP is the clear leader in the open AI tool protocol space as of early 2026. Revisit at Phase 3.

**5. API key handling:** All LLM API calls are proxied through the Theia backend service layer. No API keys reach the browser.

**6. LLM providers:** Claude Sonnet 4.6 as the primary model. Ollama (local) as the fallback for air-gapped deployments.

---

## Consequences

**Easier:**
- Theia AI's UI contributions (ChatWidget, PromptTemplate management) reduce custom UI work
- MCP server enables external AI tooling without any additional API development
- Single tool implementation serves both Theia AI agents and external MCP clients
- Anthropic's Claude is the primary LLM — aligns with existing `@clark/ai` investment

**Harder:**
- Theia AI framework requires Theia 1.54+ — needs verification that Anthropic provider contribution exists or must be built
- MCP server adds another package to maintain
- The `MCPToolProvider` bridge adds one layer of indirection to debug
- Prompt governance in production: system prompt changes must be versioned and reviewed — add to QA process

**Superseded by:** Nothing yet. Revisit MCP vs emerging protocols at Phase 3 planning.
