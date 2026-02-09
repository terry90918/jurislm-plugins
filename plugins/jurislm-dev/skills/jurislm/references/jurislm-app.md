# jurislm_app Architecture Reference

Complete architecture reference for the JurisLM Web Application.

## Overview

| Attribute | Value |
|-----------|-------|
| Framework | Next.js 16 + React 19 |
| Runtime | Bun |
| UI Library | shadcn/ui + Radix UI |
| ORM | Drizzle ORM 0.45 |
| Styling | Tailwind CSS 4 |
| AI Framework | Anthropic SDK (Unified Agent) |
| Port | 3000 |

## Directory Structure

```
jurislm_app/
+-- app/                            # Next.js App Router
|   +-- (app)/                      # Route group (requires auth)
|   |   +-- page.tsx                # Dashboard
|   |   +-- chats/                  # Conversation list
|   |   +-- knowledge/              # Knowledge base management
|   |   +-- projects/               # Project management
|   |   +-- settings/               # Settings (connectors etc.)
|   +-- c/[uuid]/                   # Conversation page (UUID route)
|   +-- auth/                       # Auth pages (signin, signout, error)
|   +-- api/                        # API Routes
|   |   +-- chat/                   # Chat streaming endpoint
|   |   +-- conversations/          # Conversation CRUD
|   |   +-- knowledge/              # Knowledge base CRUD
|   |   +-- connectors/             # Google Drive OAuth
|   |   +-- projects/               # Project management
|   |   +-- sources/                # Source preview
|   |   +-- health/                 # Health check
|   +-- layout.tsx                  # Root layout
|   +-- globals.css                 # Global styles
|
+-- lib/                            # Core business logic
|   +-- logging.ts                  # App-level logger factory
|   +-- agents/                     # Unified Agent
|   |   +-- unified/                # Core agent implementation
|   |   |   +-- agent.ts            # Agentic loop (runUnifiedAgent, max 15 iterations)
|   |   |   +-- SKILL.md            # Agent behavior definition (system prompt)
|   |   |   +-- tool-registry.ts    # 11 Anthropic Tool definitions + executeTool dispatcher
|   |   |   +-- tools/              # 5 tool implementation files
|   |   |       +-- search-knowledge.ts   # RAG search (shared + personal + project)
|   |   |       +-- contract-tools.ts     # parse_contract, analyze_clause, render_contract_report
|   |   |       +-- nda-tools.ts          # parse_nda, check_nda, render_nda_report
|   |   |       +-- document-tools.ts     # extract_document_info, generate_legal_document
|   |   |       +-- sanitize.ts           # sanitize_content
|   |   +-- shared/                 # Shared agent modules
|   |       +-- classification.ts   # Classification types and conversion
|   |       +-- llm-factory.ts      # LLM factory (Anthropic SDK)
|   |       +-- prompts.ts          # Shared prompt templates
|   +-- connectors/                 # External connectors
|   |   +-- google-drive.ts         # Google Drive OAuth
|   +-- knowledge/                  # Knowledge base module
|   |   +-- state-machine.ts        # Document processing state machine
|   |   +-- processor.ts            # Document processor
|   |   +-- storage.ts              # Local storage
|   +-- risk/                       # Risk assessment
|   |   +-- matrix.ts               # 5x5 risk matrix
|   +-- playbook/                   # Playbook loader
|   +-- langfuse/                   # LLM Observability
|   |   +-- client.ts               # Langfuse client (Singleton)
|   +-- db/                         # Drizzle ORM
|   |   +-- schema.ts               # Database schema
|   |   +-- queries.ts              # Query functions
|   +-- embedding/                  # Embedding utilities
|   +-- utils.ts                    # Utility functions
|
+-- components/
|   +-- chat/                       # Chat interface components
|   +-- layout/                     # Layout components (sidebar etc.)
|   +-- ui/                         # shadcn/ui components
|
+-- hooks/                          # React hooks
+-- public/                         # Static assets
+-- tests/                          # Vitest unit tests
|
+-- package.json
+-- next.config.ts
+-- tsconfig.json
+-- vitest.config.ts
+-- components.json                 # shadcn/ui config
```

## Unified Agent Architecture

### Overview

1 Agent + SKILL.md + 11 Tools replaces the previous 4 LangGraph Agents + keyword router:

- **SKILL.md** serves as the system prompt defining agent behavior
- **tool-registry.ts** defines 11 Anthropic Tool definitions + `executeTool` dispatcher
- **agent.ts** implements the agentic loop: LLM → tool_use → executeTool → tool_result → LLM → ... → text response
- Agent type is detected from the first tool call (parse_contract→contract, parse_nda→nda, etc.)

### Agentic Loop (agent.ts)

```
User Query → runUnifiedAgent()
  → Build system prompt (SKILL.md + context)
  → Loop (max 15 iterations):
      → Call Anthropic API with tools
      → If response contains tool_use:
          → executeTool(toolName, input)
          → Append tool_result to messages
          → SSE: step_start/step_end events
          → Continue loop
      → If response is text:
          → SSE: content event
          → Break loop
  → SSE: done event
  → Save messages to DB
```

### Tool Registry (tool-registry.ts)

11 tools organized into 4 categories:

| Category | Tools | Implementation File |
|----------|-------|-------------------|
| RAG | `search_knowledge` | `tools/search-knowledge.ts` |
| Contract | `parse_contract`, `analyze_clause`, `render_contract_report`, `load_playbook` | `tools/contract-tools.ts` |
| NDA | `parse_nda`, `check_nda`, `render_nda_report` | `tools/nda-tools.ts` |
| Document | `extract_document_info`, `generate_legal_document` | `tools/document-tools.ts` |
| Security | `sanitize_content` | `tools/sanitize.ts` |

### Agent Type Detection

Agent type is auto-detected from the first tool call:

| First Tool Call | Agent Type | SSE `agent_type` |
|----------------|------------|-------------------|
| `parse_nda` | nda | `nda` |
| `parse_contract` | contract | `contract` |
| `extract_document_info` | document | `document` |
| `search_knowledge` | rag | `rag` |

Detection uses a config-driven `TOOL_TO_AGENT_TYPE` map (not a switch statement).

### SSE Event Types (10 types)

| Event | Data | Purpose |
|-------|------|---------|
| `agent_type` | `{ type: "rag" \| "contract" \| "nda" \| "document" }` | Agent type detection |
| `step_start` | `{ tool: string, input: object }` | Tool execution started |
| `step_end` | `{ tool: string, result: object }` | Tool execution completed |
| `content` | `{ text: string }` | Streaming text content |
| `source` | `{ citations: SourceCitation[] }` | Source citations |
| `contract_result` | `{ report: ContractReport }` | Contract analysis result |
| `nda_result` | `{ report: NDAReport }` | NDA triage result |
| `document_question` | `{ question: string }` | Document clarification question |
| `document_result` | `{ document: GeneratedDocument }` | Generated legal document |
| `done` | `{}` | Stream completed |
| `error` | `{ message: string }` | Error occurred |

### LLM Factory (lib/agents/shared/llm-factory.ts)

Creates Anthropic SDK client instances:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { getModelId, validateAnthropicApiKey } from "@jurislm/llm-config";

export function createAnthropicClient(): Anthropic {
  const apiKey = validateAnthropicApiKey();
  return new Anthropic({ apiKey });
}
```

### Key Concepts

**Three-layer Knowledge Base**:
- Without project binding: shared DB (judicial + law) + personal documents
- With project binding: shared DB + personal documents + project documents

**SourceCitation type** (exported from `unified/tools/search-knowledge.ts`):
```typescript
interface SourceCitation {
  id: string;
  type: "law" | "judgment" | "document";
  title: string;
  summary: string;
  score: number;
  metadata: Record<string, unknown>;
}
```

## UI: Three-Panel Layout

```
+----------+ +------------------------+ +--------+
| Sidebar  | |     Chat View          | |Context |
|  250px   | |  Messages + SSE Stream | | Panel  |
|          | |  Thinking Steps        | | 400px  |
|          | | +--------------------+ | |        |
|          | | | Input + Model Sel  | | |        |
|          | | +--------------------+ | |        |
+----------+ +------------------------+ +--------+
```

### Context Panel Types

| Type | Trigger | Content |
|------|---------|---------|
| `contract` | `contract_result` event | Clause analysis with GREEN/YELLOW/RED |
| `nda` | `nda_result` event | 10-point NDA checklist |
| `document` | `document_result` event | Generated document preview |
| `source` | Click on citation | Law/judgment detail view |
| `flow` | Toggle button | Agent execution diagram |

### Chat Data Flow

```
Input → POST /api/chat (SSE) → auth → lock → runUnifiedAgent()
  → LLM ↔ tool_use loop → SSE events → frontend update → save msg
```

## API Routes

### Chat Endpoint

```
POST /api/chat
Content-Type: application/json

{
  "message": "string",
  "conversationId": "string?",
  "model": "haiku" | "sonnet",
  "thinkMode": boolean,
  "thinkingBudgetTokens": number,
  "projectId": "string?",
  "attachments": Attachment[]
}

Response: SSE stream (10 event types)
```

### Conversations

```
GET    /api/conversations              # List conversations
POST   /api/conversations              # Create conversation
GET    /api/conversations/:id          # Get conversation
DELETE /api/conversations/:id          # Delete conversation
GET    /api/conversations/:id/messages # Get messages
```

### Projects

```
GET    /api/projects                   # List projects
POST   /api/projects                   # Create project
GET    /api/projects/:id               # Get project
PATCH  /api/projects/:id               # Update project
DELETE /api/projects/:id               # Delete project
GET    /api/projects/:id/documents     # Project documents
POST   /api/projects/:id/documents     # Upload document
```

### Health Check

```
GET /api/health
Response: { status: "healthy", timestamp: "ISO8601" }
```

## Database Integration

### Drizzle ORM Schema

```typescript
// lib/db/schema.ts
import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core';

export const conversations = pgTable('conversations', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id),
  title: text('title'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

export const messages = pgTable('messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  conversationId: uuid('conversation_id').references(() => conversations.id),
  role: text('role').notNull(), // 'user' | 'assistant'
  content: text('content').notNull(),
  metadata: jsonb('metadata'),  // Cross-turn state storage
  citations: jsonb('citations'),
  createdAt: timestamp('created_at').defaultNow(),
});
```

**Note**: `conversations` table has NO `metadata` column. Use `messages.metadata` for cross-turn state.

### Vector Search

Vector search queries against jurislm_shared_db:

```typescript
// Example: Law vector search
const results = await db.execute(sql`
  SELECT
    l.pcode,
    l.law_name,
    le.embedding <=> ${queryEmbedding} as distance
  FROM law_embeddings le
  JOIN laws l ON le.law_id = l.id
  ORDER BY distance
  LIMIT 10
`);
```

## Environment Variables

```bash
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5433/jurislm_db
SHARED_DATABASE_URL=postgresql://postgres:postgres@localhost:5440/jurislm_shared_db

# Authentication
GOOGLE_CLIENT_ID=xxx
GOOGLE_CLIENT_SECRET=xxx
NEXTAUTH_SECRET=xxx
NEXTAUTH_URL=http://localhost:3000

# LLM (uses @jurislm/llm-config for model-to-provider mapping)
ANTHROPIC_API_KEY=sk-ant-xxx
OLLAMA_MODEL=ministral-3:latest
OLLAMA_BASE_URL=http://localhost:11434

# Embedding (ollama = primary, tei = backup)
EMBEDDING_PROVIDER=ollama           # Options: ollama | tei
EMBEDDING_URL=http://localhost:11434  # Ollama default

# Observability (Langfuse - optional, tracing disabled if not set)
LANGFUSE_SECRET_KEY=sk-lf-xxx
LANGFUSE_PUBLIC_KEY=pk-lf-xxx
LANGFUSE_HOST=https://cloud.langfuse.com
```

## Development Commands

```bash
cd jurislm_app

# Development
bun run dev                    # Start dev server (port 3000)

# Testing
bun run test                   # Run Vitest tests
bun run test:coverage          # With coverage

# Build
bun run build                  # Production build
bun run start                  # Start production server

# Linting
bun run lint                   # ESLint
bun run typecheck              # TypeScript check
```

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| next | 16.x | React framework |
| react | 19.x | UI library |
| drizzle-orm | 0.45.x | ORM |
| @anthropic-ai/sdk | latest | Anthropic API (Unified Agent) |
| @radix-ui/* | latest | UI primitives |
| tailwindcss | 4.x | CSS framework |
| vitest | latest | Testing |

## Testing

### Unit Tests

```typescript
// tests/unit/agents/unified/tool-registry.test.ts
import { describe, it, expect } from 'vitest';
import { executeTool } from '@/lib/agents/unified/tool-registry';

describe('executeTool', () => {
  it('should dispatch to correct tool implementation', async () => {
    const result = await executeTool('sanitize_content', {
      content: 'test <script>alert("xss")</script>'
    });
    expect(result).not.toContain('<script>');
  });
});
```

## Gotchas

- `searchUserDocuments` takes options object `{userId, scope, projectId, topK, minScore}` not positional args
- `renderNDATriageReport` interface: `{title, summary, classification, checklistResults, recommendedAction}`
- Contract `parseContract` returns `{metadata, clauses}`, not just clauses
- `conversations` table has NO metadata column; use `messages.metadata` for cross-turn state
- `SourceCitation` type exported from `unified/tools/search-knowledge` (not from rag/state)
