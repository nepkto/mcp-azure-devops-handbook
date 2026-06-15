# Section 1: MCP Foundations

> **Role:** Senior Software Architect / AI Systems Expert
> **Project:** MCP Server for Azure DevOps PR Review

---

## ✅ Summary

Model Context Protocol (MCP) is an **open standard** that defines how LLMs connect to external tools, data sources, and services in a **decoupled, portable, and standardized** way. It solves the core problem of LLMs being isolated — unable to access real-time data or take real-world actions.

---

## ✅ Key Concepts

- **LLM (Copilot):** Understands intent, selects tools, interprets results, generates responses
- **MCP Client:** Lives in the IDE/app (e.g., VS Code). Routes tool calls. You do NOT build this.
- **MCP Server:** YOUR code. Exposes tools. Calls external APIs. Returns structured JSON.
- **Tool Discovery:** MCP Servers advertise their tools — the LLM discovers them at runtime (no hardcoding)
- **Transport:** MCP communicates via JSON-RPC over stdio (local) or HTTP/SSE (remote)
- **Standard Protocol:** Like USB-C — one standard, works across any compliant LLM client

---

## ✅ Architecture Diagram

```
USER
  │  "Review my PR #42 for code quality"
  ▼
LLM (GitHub Copilot)
  │  Decides: call get_pr_details({id: 42})
  ▼
MCP CLIENT (VS Code - built-in)
  │  MCP Protocol (JSON-RPC)
  ▼
MCP SERVER (Your Node.js code)
  │  HTTPS / REST
  ▼
AZURE DEVOPS (PR data, diffs, comments)
```

---

## ✅ Comparison Table

| Feature           | REST API | Function Calling | MCP         |
|-------------------|----------|-----------------|-------------|
| Who calls it?     | App code | LLM decides     | LLM decides |
| Standardized?     | ❌       | Partially        | ✅ Yes      |
| Portable?         | ❌       | ❌ Vendor-locked | ✅ Yes      |
| LLM-aware schema? | ❌       | ✅               | ✅ Richer   |
| Decoupled?        | ❌       | ❌               | ✅ Yes      |
| Auto-discoverable?| ❌       | ❌               | ✅ Yes      |

---

## ✅ Code Snippets

### Minimal MCP Server (Node.js) — Conceptual Preview
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "azure-devops-pr-review",
  version: "1.0.0",
});

// Register a tool
server.tool(
  "get_pr_details",                          // Tool name
  "Fetches details of a Pull Request",       // Description (LLM reads this!)
  { pr_id: z.number() },                    // Input schema
  async ({ pr_id }) => {
    // Call Azure DevOps API here
    return { content: [{ type: "text", text: JSON.stringify(prData) }] };
  }
);

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## ✅ Design Guidelines / Best Practices

- **Tool descriptions are critical** — the LLM uses them to decide which tool to call. Be clear and specific.
- **Keep MCP Server stateless** where possible — easier to scale and debug
- **One responsibility per tool** — `get_pr_details` should only get PR details, not also post comments
- **Return structured JSON** — LLMs consume structured data better than unstructured text
- **Log everything** — MCP servers run as subprocesses; logging is your only debug window

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| Vague tool descriptions | LLM picks wrong tool or doesn't use it at all |
| Combining too many operations in one tool | Hard for LLM to know when to use it |
| Returning unstructured text blobs | LLM struggles to extract meaningful data |
| Hardcoding tool logic in the client | Defeats the purpose of MCP's decoupling |
| Not handling errors gracefully | LLM gets confused with unexpected output |

---

## ✅ Azure DevOps PR Example

**User says:** *"Review PR #42 and check if there are any security issues"*

**What happens:**
1. Copilot reads the MCP Server's tool list
2. Copilot identifies `get_pr_diff` as the right tool
3. MCP Client calls `get_pr_diff({ pr_id: 42 })`
4. MCP Server calls Azure DevOps REST API
5. Returns structured diff data
6. Copilot analyzes the diff and produces a security review

---

## ✅ How I Will Use This in My MCP Project

- **I am building the MCP Server** — VS Code + Copilot is already the MCP Client
- My server will expose tools specifically for Azure DevOps PR review
- Tools I plan to build:
  - `get_pr_details` — fetch PR metadata
  - `get_pr_diff` — fetch changed files and diffs
  - `get_pr_comments` — fetch existing review comments
  - `post_review_comment` — post AI-generated feedback
  - `review_pr` *(advanced)* — orchestrates all of the above
- The LLM will discover and call these tools automatically based on user prompts

---

*Next: Section 2 — Architecture Deep Dive*