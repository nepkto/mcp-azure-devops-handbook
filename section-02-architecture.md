# Section 2: Architecture Deep Dive

---

## ✅ Summary

MCP follows a strict **3-layer architecture**: LLM → MCP Client → MCP Server → External API.
The LLM selects tools based on **tool descriptions** (not hardcoded logic). The MCP Client handles
transport. The MCP Server handles business logic and external API calls. Understanding each layer's
responsibility is essential before writing a single line of code.

---

## ✅ Full Request Flow

```
User
  │  "Review PR #42 for security issues"
  ▼
LLM (GitHub Copilot)
  │  Reads tool list → Reasons → Picks "get_pr_diff"
  │  Generates: { name: "get_pr_diff", args: { pr_id: 42 } }
  ▼
MCP Client (VS Code - built-in)
  │  Serializes as JSON-RPC
  │  Sends to MCP Server via stdio or HTTP
  ▼
MCP Server (Your Node.js / Python code)
  │  Validates input → Calls Azure DevOps API → Formats response
  ▼
Azure DevOps
  │  Returns PR diff, files, metadata
  ▼
MCP Server → MCP Client → LLM
  │  LLM analyzes → Generates review → User sees result
  ▼
User
  "PR #42 has 3 security issues: SQL injection on line 42..."
```

---

## ✅ Key Concepts

- **Tool Discovery:** MCP Client asks MCP Server for its tool list at startup — LLM sees this list
- **Tool Selection:** LLM reads tool `name` + `description` to decide which tool matches user intent
- **JSON-RPC:** The wire protocol MCP uses — lightweight, structured, bidirectional
- **Transport Options:**
  - `stdio` — Local subprocess (simplest, good for development)
  - `HTTP/SSE` — Remote server (good for production/shared deployments)
- **Input Schema:** Zod (Node.js) or JSON Schema (Python) — validates and describes tool inputs
- **Tool Result:** Always returns `{ content: [{ type: "text", text: "..." }] }`

---

## ✅ Responsibility Table

### MCP Client (Built-in — VS Code)
| Responsibility     | Detail                                          |
|--------------------|-------------------------------------------------|
| Tool Discovery     | Fetches tool list from server at startup        |
| Routing            | Knows which server handles which tool           |
| Lifecycle          | Starts/stops MCP Server process                 |
| Transport          | Speaks JSON-RPC over stdio or HTTP/SSE          |

### MCP Server (You Build This)
| Responsibility     | Detail                                          |
|--------------------|-------------------------------------------------|
| Tool Registration  | Declares tools with name, description, schema  |
| Tool Execution     | Runs actual logic when called                   |
| API Integration    | Calls Azure DevOps, handles auth (PAT)          |
| Response Format    | Returns structured, LLM-friendly JSON           |
| Error Handling     | Returns meaningful errors — not stack traces    |

---

## ✅ Code Snippets

### Node.js (TypeScript) — Tool Registration

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "azure-devops-pr-review",
  version: "1.0.0",
});

server.tool(
  "get_pr_diff",
  `Fetches the file changes and code diff for a Pull Request in Azure DevOps.
   Use this when the user wants to review code, check changes, or analyze modifications.`,
  {
    pr_id: z.number().describe("The Pull Request ID number"),
    repository: z.string().describe("The repository name in Azure DevOps"),
    file_filter: z.string().optional().describe("Optional file extension filter, e.g. '.ts'"),
  },
  async ({ pr_id, repository, file_filter }) => {
    const diff = await fetchPRDiff(pr_id, repository, file_filter);
    return { content: [{ type: "text", text: JSON.stringify(diff, null, 2) }] };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Python — Tool Registration

```python
import asyncio, json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("azure-devops-pr-review")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_pr_diff",
            description="""Fetches the file changes and code diff for a Pull Request in Azure DevOps.
            Use this when the user wants to review code, check changes, or analyze modifications.""",
            inputSchema={
                "type": "object",
                "properties": {
                    "pr_id": {"type": "integer", "description": "The Pull Request ID number"},
                    "repository": {"type": "string", "description": "The repository name in Azure DevOps"},
                    "file_filter": {"type": "string", "description": "Optional file extension filter"}
                },
                "required": ["pr_id", "repository"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "get_pr_diff":
        diff = await fetch_pr_diff(arguments["pr_id"], arguments["repository"])
        return [TextContent(type="text", text=json.dumps(diff, indent=2))]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as streams:
        await server.run(streams[0], streams[1], server.create_initialization_options())

asyncio.run(main())
```

---

## ✅ Design Guidelines / Best Practices

- **Tool descriptions are your API contract with the LLM** — invest time here
- **Use `describe()` on every schema field** — LLM uses these to extract values from user input
- **Start with stdio transport** — simple, no networking, easier to debug
- **One MCP Server per domain** — keep your Azure DevOps server focused
- **Return consistent output shapes** — LLM gets confused by varying structures
- **Always handle the case where Azure DevOps API returns an error**

---

## ✅ Common Mistakes

| Mistake                              | Why It Fails                                          |
|--------------------------------------|-------------------------------------------------------|
| Short/vague tool description         | LLM won't know when or how to use the tool            |
| Missing `describe()` on schema fields| LLM can't extract correct argument values             |
| Returning HTML or raw API responses  | LLM can't parse unstructured blobs efficiently        |
| Building logic into the MCP Client   | Breaks portability and defeats MCP's purpose          |
| No error handling in tool executor   | LLM receives a crash/exception — unpredictable output |

---

## ✅ Azure DevOps PR Example

**Tool Schema the LLM sees:**
```json
{
  "name": "get_pr_diff",
  "description": "Fetches file changes and code diffs for a PR. Use when reviewing code changes.",
  "inputSchema": {
    "pr_id": "integer — The PR number",
    "repository": "string — Repo name",
    "file_filter": "string (optional) — Filter by file extension"
  }
}
```

**User says:** *"Check PR #42 for any TypeScript issues"*

**LLM reasons:**
1. Intent = review TypeScript code in PR
2. Tool `get_pr_diff` matches → call with `{ pr_id: 42, file_filter: ".ts" }`
3. Receives diff → analyzes → reports TypeScript-specific issues

---

## ✅ How I Will Use This in My MCP Project

- I now understand the **exact flow** from user prompt → Azure DevOps API → LLM response
- My MCP Server will:
  - Use **stdio transport** during development
  - Register tools with **rich descriptions** so Copilot selects them correctly
  - Return **structured JSON** (not raw API responses)
  - Validate all inputs with **Zod** (Node.js) or **JSON Schema** (Python)
- I will NOT put any business logic in the MCP Client side

---

*Next: Section 3 — MCP Server Design (Tool naming, schema design, good vs bad examples)*