+++
title = "Deconstructing MCP: What It Is, What It Solves, and How the Pieces Fit"
date = '2026-09-18'
draft = false
tags = ['mcp', 'architecture', 'llm', 'tool-use', 'typescript']
categories = ['AI']
+++

Model Context Protocol (MCP) has become the standard way we connect LLMs to external systems. But if you read the official specs or marketing material, it’s easy to get lost in buzzwords or assume it’s some magical orchestrator that does everything.

Strip the hype away, and MCP is straightforward: it is a **protocol**. It does what it says on the tin. It passes payloads back and forth according to a standardized schema, giving our models and tools a common language to speak.

Let's break down the mental model, how the end-to-end request flow actually works, and where the boundaries should live in a real codebase.

---

### The Big Picture: Where MCP Sits

Suppose you are building an AI-powered system with four main layers:

```text
+------+        +------+        +------+        +------------+
| web/ | -----> | api/ | -----> | mcp/ | -----> | tools/     |
| (UI) |        |      |        |      |        | (impl)     |
+------+        +------+        +------+        +------------+
```

Here is how each layer behaves:

1. **`web/` (UI):** A user in your React/web frontend submits an action or prompt.
2. **`api/` (Orchestration):** Handles the user session, manages conversation history, talks to the LLM provider (OpenAI, Anthropic, local models), and hosts the MCP *client*.
3. **`mcp/` (Protocol & Transport):** The MCP *server*. It acts as a thin transport layer. It speaks JSON-RPC, validates arguments against schemas, and exposes standardized endpoints like `tools/list` and `tools/call`.
4. **`tools/` (Implementation):** The actual business logic — reading files, executing SQL queries, calling external APIs, or modifying state.

---

### The Request Lifecycle

When a user asks the model to perform an action, the sequence across these layers is clear and mechanical:

1. **Discovery:** `api/` queries `mcp/` for available tools via `tools/list`.
2. **LLM Prompting:** `api/` attaches those tool definitions (name, description, input schema) to the LLM completion request.
3. **Decision:** The LLM decides to trigger an action and responds with a structured `tool_call` containing the tool name and arguments.
4. **Dispatch:** `api/` receives the tool call and forwards it to `mcp/` via `tools/call`.
5. **Execution:** `mcp/` routes the request to `tools/` to run the concrete logic, formats the return value according to MCP rules, and relays it back up to `api/`.

```text
[User] -> [web/] -> [api/] ------------ (tools/list) ------------> [mcp/] -> [tools/]
                      |                                               |
                      +-- (Prompt + Tool Schemas) --> [ LLM ]         |
                      |                                  |            |
                      +<-- (tool_call: name, args) ------+            |
                      |                                               |
                      +------------------ (tools/call) -------------->+
                                                                      |
                                                                  [Run Tool]
```

A common point of confusion is: **When does the LLM get the tool list?**

The LLM provider itself does not know about your MCP server. MCP lives strictly between your `api/` and `mcp/` layers. Tool definitions are not magically cached inside the model between API calls; your `api/` must attach the tool definitions to every single request that expects tool availability.

---

### The Unified Tool Contract: "Execute"

To keep MCP decoupled from domain logic, `mcp/` should never know *how* a tool works. Whether a tool reads a file or queries PostgreSQL, it adheres to the same universal interface:

```typescript
interface ToolHandler {
  name: string;
  description: string;
  schema: Record<string, unknown>; // JSON Schema for arguments
  execute(args: unknown): Promise<ToolResult>;
}
```

Every tool implements this contract. `mcp/` only calls `.execute(args)` and passes the result back. This ensures all actors in the network speak the same structural language regardless of the underlying business rules.

---

### Single Source of Truth: Cache, Don't Own

Where should the list of tools actually live?

It is tempting to write a hardcoded registry inside `mcp/`. That is an anti-pattern bound to cause schema drift. If someone updates `edit_file` to require a new parameter but forgets to update `mcp/`, things break silently.

The golden rule is: **`tools/` owns the schemas and implementations; `mcp/` asks and caches.**

```text
tools/  --- (owns schemas & impl) --->  mcp/  --- (caches what tools/ provided) --->  api/
```

#### Scenario 1: Same Language, In-Process (Node / TypeScript)

If your MCP server and tools share the same runtime, `tools/` exports its self-describing modules, and `mcp/` imports them directly at boot:

```typescript
// tools/edit_file.ts
export const editFileTool: ToolHandler = {
  name: "edit_file",
  description: "Edits a file safely with an exact patch",
  schema: editFileSchema,
  execute: editFileHandler,
};

// tools/index.ts
export const allTools: ToolHandler[] = [editFileTool, readFileTool];
```

Inside your MCP server (for example, using Fastify or Express):

```typescript
// mcp/server.ts
import { allTools } from "../tools/index.js";

const toolCache = new Map(allTools.map(t => [t.name, t]));

// MCP tools/list handler
server.post("/tools/list", async () => {
  return Array.from(toolCache.values()).map(({ name, description, schema }) => ({
    name,
    description,
    schema,
  }));
});

// MCP tools/call handler
server.post("/tools/call", async (req) => {
  const { name, args } = req.body;
  const tool = toolCache.get(name);
  if (!tool) throw new Error(`Unknown tool: ${name}`);
  return await tool.execute(args);
});
```

`mcp/` builds its cache on startup by inspecting what `tools/` exported. If a tool definition changes, restarting or reloading updates the schema automatically.

#### Scenario 2: Cross-Language Services (e.g. Node MCP + Go Tools)

If your tools are written in Go, Rust, or Python, Node cannot import them directly. In that architecture, `tools/` runs as a dedicated microservice exposing discovery and execution endpoints:

```text
GET  /tools          -> returns [{ name, description, schema }, ...]
POST /tools/execute  -> { name: "edit_file", args: {...} } -> { result }
```

`mcp/` acts as an HTTP/gRPC client to the tools service, fetching the tool definitions with a TTL cache and proxying execution requests:

```typescript
// mcp/registry.ts
let toolCache: ToolDefinition[] = [];
let lastFetched = 0;
const TTL_MS = 60_000;

async function getTools(): Promise<ToolDefinition[]> {
  if (Date.now() - lastFetched < TTL_MS) {
    return toolCache;
  }
  const res = await fetch(`${TOOLS_SERVICE_URL}/tools`);
  toolCache = await res.json();
  lastFetched = Date.now();
  return toolCache;
}
```

---

### Anatomy of an MCP Server

A well-structured `mcp/` service remains lightweight and focused on transport responsibilities:

```text
mcp/
├── server.ts            # Entrypoint & transport setup (stdio, HTTP, SSE)
├── registry.ts          # Discovery & caching layer for tool definitions
├── handlers/
│   ├── listTools.ts     # Implements tools/list per specification
│   └── callTool.ts      # Implements tools/call and validates inputs
├── schema/
│   └── jsonrpc.ts       # Wire envelope validation
└── errors.ts            # Normalizes execution errors into MCP error responses
```

---

### What to Read Next

- For an example of skipping HTTP overhead entirely during MVP prototyping, check out [No HTTP, No Hacks: In-Process MCP with the Node SDK](https://edchtzwe.github.io/blog/posts/ai/01-mcp-in-memory-transport/).
