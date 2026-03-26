# Framework: AI, LLM Communication, and MCP Integration

**Covers:** Theia AI framework architecture, LLM provider configuration (Claude/OpenAI/Ollama), Model Context Protocol (MCP) server design, Theia AI ↔ MCP adapter, AI audit trail, OpenClaw and emerging AI tool protocols.

---

## 1. Theia AI Architecture

Theia AI (introduced 2024) is an extensible AI framework built into Theia. It is not a standalone product — it is a set of contribution points and services that any Theia extension can use.

### Core contribution points

| Contribution | Interface | Purpose |
|-------------|-----------|---------|
| `AIAgent` | `Agent` | Define a named AI agent with system prompt, tools, and LLM provider |
| `PromptTemplate` | `PromptTemplate` | Register named, parameterizable prompts managed in Theia Preferences |
| `ToolProvider` | `ToolProvider` | Expose IPE services as callable tools for AI agents |
| `LanguageModelProvider` | `LanguageModelProvider` | Register an LLM backend (OpenAI, Claude, Ollama) |
| `ChatWidget` | `AbstractViewContribution` | Embed a chat UI in any Theia panel |
| `AIVariableService` | `AIVariableContribution` | Register runtime variables injected into prompts |

### Theia AI data flow

```
Operator types prompt in ChatWidget
  ↓
Theia AI resolves variables (job ID, workstation, operator name)
  ↓
Resolved prompt → AIAgent
  ↓
AIAgent → LanguageModelProvider → LLM API (Claude/OpenAI/Ollama)
  ↓
LLM response → ToolProvider (if tool call)
  ↓
ToolProvider → IPE service (JobService, InspectionService, etc.)
  ↓
Tool result → back to LLM for response synthesis
  ↓
Final response → ChatWidget + AuditLogger
```

---

## 2. LLM Provider Configuration

### Claude (Anthropic) via native Theia AI provider

Theia AI has an Anthropic provider contribution (being upstreamed as of 2025). Configuration in Theia Preferences:

```json
{
  "ai-core.anthropic.apiKey": "${env:ANTHROPIC_API_KEY}",
  "ai-core.anthropic.models": [
    { "id": "claude-sonnet-4-6", "name": "Claude Sonnet 4.6 (Default)" },
    { "id": "claude-haiku-4-5-20251001", "name": "Claude Haiku 4.5 (Fast)" }
  ]
}
```

**API key handling:** The API key must be stored in the Theia backend process environment — never in the browser. The `AnthropicLanguageModelProvider` is a `BackendApplicationContribution` that reads the key from the process env and proxies LLM calls via the Theia JSON-RPC channel to the frontend.

### OpenAI-compatible endpoints

Theia AI's OpenAI provider works with any OpenAI-compatible API:

```json
{
  "ai-core.openai.apiKey": "${env:OPENAI_API_KEY}",
  "ai-core.openai.models": [
    { "id": "gpt-4o", "name": "GPT-4o" }
  ]
}
```

For Azure OpenAI or a custom endpoint:
```json
{
  "ai-core.openai.baseUrl": "https://my-azure-endpoint.openai.azure.com/",
  "ai-core.openai.apiKey": "${env:AZURE_OPENAI_KEY}"
}
```

### Ollama (local/air-gapped deployment)

Ollama runs a local LLM server with an OpenAI-compatible API:

```bash
docker run -d --gpus all -p 11434:11434 ollama/ollama
ollama pull llama3.1:8b  # or mistral, phi3, etc.
```

Configuration in Theia:
```json
{
  "ai-core.openai.baseUrl": "http://localhost:11434/v1",
  "ai-core.openai.apiKey": "ollama"
}
```

**Air-gapped deployment consideration:** For defense-adjacent customers who cannot allow external LLM calls, Ollama on the node server (with a quantized model like `mistral-7b` or `llama3.1-8b`) provides a fully local AI capability. The model quality is lower than Claude but sufficient for structured tasks like defect classification and job summarization.

---

## 3. Model Context Protocol (MCP)

### What MCP is

MCP (Model Context Protocol) is an open protocol by Anthropic (November 2024) that standardizes how AI systems communicate with tools and data sources. An MCP server exposes:
- **Resources** — Data the LLM can read (files, database records, live state)
- **Tools** — Functions the LLM can call (write operations, queries, actions)
- **Prompts** — Reusable prompt templates with parameters

### Clark IPE MCP Server

The Clark IPE exposes an MCP server that gives AI agents (Claude, external agents, customer integrations) programmatic access to IPE services.

**Package:** `packages/mcp-server/` — a standalone Node.js MCP server that runs alongside the Fastify API.

**Transport:** MCP supports two transports:
- **stdio** — for local processes (Claude Desktop, Theia AI backend calling a local MCP process)
- **SSE (Server-Sent Events)** — for remote HTTP clients

For Theia AI integration, the MCP server runs as a local process and communicates via stdio.

### MCP server resource definitions

```typescript
// packages/mcp-server/src/resources.ts

const resources: MCPResource[] = [
  {
    uri: 'clark://jobs/{jobId}',
    name: 'Job Record',
    description: 'Full job record including status, tasks, notes, and events',
    mimeType: 'application/json'
  },
  {
    uri: 'clark://jobs/{jobId}/inspection',
    name: 'Inspection Record',
    description: 'All inspection points and results for a job',
    mimeType: 'application/json'
  },
  {
    uri: 'clark://jobs/{jobId}/audit-log',
    name: 'Job Audit Log',
    description: 'Chronological event log for a job',
    mimeType: 'application/json'
  },
  {
    uri: 'clark://workstation/{wsId}/active-job',
    name: 'Active Job at Workstation',
    description: 'The currently active job at this workstation',
    mimeType: 'application/json'
  },
  {
    uri: 'clark://ipc-criteria/{class}/{stepType}',
    name: 'IPC Workmanship Criteria',
    description: 'IPC-A-610 criteria for a given assembly class and step type',
    mimeType: 'application/json'
  }
];
```

### MCP server tool definitions

```typescript
const tools: MCPTool[] = [
  {
    name: 'create_note',
    description: 'Create a note on a job with AI-generated content. Note will be labeled as AI-generated.',
    inputSchema: {
      type: 'object',
      properties: {
        jobId: { type: 'string' },
        content: { type: 'string' },
        noteType: { enum: ['observation', 'instruction', 'anomaly', 'summary'] }
      },
      required: ['jobId', 'content']
    }
  },
  {
    name: 'log_inspection_result',
    description: 'Log an inspection point result. Requires operator confirmation before commit.',
    inputSchema: {
      type: 'object',
      properties: {
        jobId: { type: 'string' },
        criterionId: { type: 'string' },
        result: { enum: ['Pass', 'Fail', 'ProcessIndicator'] },
        notes: { type: 'string' }
      },
      required: ['jobId', 'criterionId', 'result']
    }
  },
  {
    name: 'classify_defect',
    description: 'Suggest IPC-A-610 defect classification from a description. Returns a suggested criterion and disposition.',
    inputSchema: {
      type: 'object',
      properties: {
        description: { type: 'string' },
        assemblyClass: { enum: ['1', '2', '3'] },
        componentType: { type: 'string' }
      },
      required: ['description', 'assemblyClass']
    }
  },
  {
    name: 'query_job_history',
    description: 'Query similar past jobs and their outcomes',
    inputSchema: {
      type: 'object',
      properties: {
        partNumber: { type: 'string' },
        assemblyClass: { type: 'string' },
        limit: { type: 'number', default: 5 }
      },
      required: ['partNumber']
    }
  },
  {
    name: 'generate_job_summary',
    description: 'Generate a structured summary of a completed or in-progress job',
    inputSchema: {
      type: 'object',
      properties: {
        jobId: { type: 'string' },
        includeDefects: { type: 'boolean', default: true },
        includeFirmware: { type: 'boolean', default: true }
      },
      required: ['jobId']
    }
  }
];
```

### Theia AI ↔ MCP Adapter

Theia AI's `ToolProvider` contribution wraps MCP tool calls:

```typescript
// apps/ipe/src/extensions/clark-ai-extension/src/node/mcp-tool-provider.ts

@injectable()
export class ClarkMCPToolProvider implements ToolProvider {
  private mcpClient: MCPClient;

  @postConstruct()
  async init(): Promise<void> {
    this.mcpClient = await MCPClient.connectStdio({
      command: 'node',
      args: [resolve(__dirname, '../../../mcp-server/dist/index.js')]
    });
  }

  getTools(): AITool[] {
    return this.mcpClient.listTools().map(tool => ({
      id: `clark.mcp.${tool.name}`,
      name: tool.name,
      description: tool.description,
      parameters: tool.inputSchema,
      handler: async (params) => this.mcpClient.callTool(tool.name, params)
    }));
  }
}
```

---

## 4. OpenClaw and Emerging AI Tool Protocols

### Context

The AI tooling ecosystem is evolving rapidly post-MCP. Relevant protocols and frameworks to monitor:

| Protocol/Tool | Status (March 2026) | Relevance to Clark |
|--------------|--------------------|--------------------|
| **MCP** | Stable v1.0, wide adoption (Claude, Cursor, Zed, Windsurf) | **Primary choice for tool integration** |
| **OpenClaw** | Emerging open-source AI interaction framework | Monitor — evaluate if MCP's HTTP/stdio transport is insufficient for real-time industrial contexts |
| **A2A (Agent-to-Agent)** | Google proposal (2025) | Relevant if Clark builds cross-node AI agent communication |
| **ACP (Agent Communication Protocol)** | IBM/BeeAI (2025) | Alternative to A2A for multi-agent orchestration |
| **LSP + AI extensions** | GitHub Copilot, Zed AI | LSP already in IPE — AI code completion available via `@theia/ai-code-completion` |

### Recommendation

**Use MCP as the primary tool protocol for Phase 1.** It is the most mature, has Anthropic's native support, has a growing ecosystem, and maps cleanly to Theia AI's ToolProvider architecture. Evaluate OpenClaw only if specific industrial protocol requirements emerge that MCP cannot satisfy (e.g., sub-10ms tool call latency on a real-time bus).

---

## 5. AI Audit Trail

### Schema

Every AI action is recorded with the following fields:

```sql
CREATE TABLE ai_audit_records (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id            TEXT,              -- may be null for non-job-scoped actions
  workstation_id    TEXT,
  actor_id          TEXT NOT NULL,     -- operator who triggered or reviewed
  agent_id          TEXT NOT NULL,     -- e.g. 'clark.inspection-assistant'
  model_id          TEXT NOT NULL,     -- e.g. 'claude-sonnet-4-6'
  prompt_template   TEXT,              -- named template used (if any)
  input_summary     TEXT,              -- truncated/summarized input (not full prompt)
  response_text     TEXT NOT NULL,
  tool_calls        JSONB,             -- array of tool calls made, if any
  review_state      TEXT NOT NULL DEFAULT 'pending',  -- pending | accepted | rejected | annotated
  review_note       TEXT,              -- operator annotation
  reviewed_by       TEXT,              -- operator actor ID
  reviewed_at       TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Audit Panel display rules

- All AI-generated content is labeled with a badge: `[AI · Claude Sonnet 4.6]`
- The operator can Accept, Reject, or Annotate any AI output
- Accepted AI outputs that create records (notes, inspection results) are written with `source: 'ai'` in the record metadata
- Rejected AI outputs are recorded as rejected — never silently discarded
- The full audit record is exportable as part of the job's compliance documentation package

---

## 6. Phase 1 AI Scope

For Phase 1 (Workstation Proof), implement only:

1. **`InspectionAssistant` agent** — Given a defect description and assembly class, suggests the applicable IPC-A-610 criterion and disposition. Uses `classify_defect` MCP tool.
2. **`JobSummarizer` agent** — On demand, generates a structured summary of a job. Uses `generate_job_summary` MCP tool.

Defer to Phase 2: AnomalyDetector, FirmwareReviewer, ShiftHandoffComposer, multi-agent orchestration.

---

## 7. Outstanding Questions / Decisions Required

- [ ] Theia AI upstream status: Is the Anthropic provider in `@theia/ai-anthropic` published, or does Clark need to implement it as a custom extension?
- [ ] MCP transport choice for Theia AI: stdio (local process) vs SSE (HTTP). Stdio is simpler and lower-latency. SSE enables external agents to connect to the Clark IPE. Recommendation: stdio for Phase 1, SSE for Phase 2.
- [ ] Ollama model selection for air-gapped: `llama3.1:8b` (8GB VRAM) or `phi3:medium` (less capable but lower hardware requirements)?
- [ ] Prompt template governance: who can edit system prompts in production? Should require a QA signoff for changes in Class 3 environments?
- [ ] AI audit record retention: how long are AI audit records kept? Are they part of the compliance package for defense-adjacent customers?
