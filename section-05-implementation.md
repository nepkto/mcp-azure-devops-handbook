# Section 5: Real-World Implementation

---

## ✅ Summary

A production-grade MCP Server needs: a clean project structure, secure PAT handling via environment
variables, a typed Azure DevOps API client, and properly registered tools with error handling.
The server wires everything together and starts via stdio transport. VS Code discovers it via
`.vscode/mcp.json`. Use the **MCP Inspector UI** to test and debug tools interactively before wiring into VS Code.

---

## ✅ Key Concepts

- **Fail Fast:** Validate all env vars at startup — crash immediately if PAT or ORG is missing
- **Single API Client:** One gateway to Azure DevOps — all tools go through it
- **PAT Auth:** Base64 encode `:<PAT>` and pass as `Basic` Authorization header
- **Error Handling:** Every tool catches exceptions and returns `{ isError: true }` — never crash the server
- **stderr for logging:** MCP uses stdout as the protocol channel — always log to `stderr`
- **`.vscode/mcp.json`:** How VS Code discovers and launches your MCP Server
- **MCP Inspector UI:** Browser-based tool to interactively call and debug your MCP tools before integrating with an LLM

---

## ✅ Project Structure

```
Node.js:                          Python:
azure-devops-mcp/                 azure-devops-mcp/
├── src/                          ├── src/
│   ├── index.ts                  │   ├── main.py
│   ├── tools/                    │   ├── tools/
│   ├── azure/devopsClient.ts     │   ├── azure/devops_client.py
│   └── utils/                    │   └── utils/
├── .env              ← secret    ├── .env              ← secret
├── .env.example      ← commit    ├── .env.example      ← commit
├── .gitignore                    ├── .gitignore
└── package.json                  └── requirements.txt
```

---

## ✅ Environment Variables

```bash
# .env (NEVER commit)
AZURE_DEVOPS_ORG=mycompany
AZURE_DEVOPS_PAT=your_pat_token_here
AZURE_DEVOPS_DEFAULT_PROJECT=MyProject

# .env.example (safe to commit)
AZURE_DEVOPS_ORG=
AZURE_DEVOPS_PAT=
AZURE_DEVOPS_DEFAULT_PROJECT=
```

---

## ✅ PAT Authentication

Azure DevOps uses HTTP Basic Auth with PAT:

```
Authorization: Basic base64(":<PAT>")
```

### Node.js
```typescript
const authHeader = `Basic ${Buffer.from(`:${PAT}`).toString("base64")}`;
```

### Python
```python
import base64
pat_encoded = base64.b64encode(f":{PAT}".encode()).decode()
headers = {"Authorization": f"Basic {pat_encoded}"}
```

---

## ✅ Code Snippets

### Node.js — Azure DevOps Client

```typescript
import axios from "axios";
import * as dotenv from "dotenv";
dotenv.config();

const ORG = process.env.AZURE_DEVOPS_ORG;
const PAT = process.env.AZURE_DEVOPS_PAT;
if (!ORG || !PAT) { console.error("[FATAL] Missing env vars"); process.exit(1); }

export const devopsClient = axios.create({
    baseURL: `https://dev.azure.com/${ORG}`,
    headers: {
        Authorization: `Basic ${Buffer.from(`:${PAT}`).toString("base64")}`,
        "Content-Type": "application/json",
    },
});

export async function fetchPRDetails(project: string, repoId: string, prId: number) {
    const res = await devopsClient.get(
        `/${project}/_apis/git/repositories/${repoId}/pullrequests/${prId}`,
        { params: { "api-version": "7.1" } }
    );
    return res.data;
}
```

### Python — Azure DevOps Client

```python
import os, base64, httpx
from dotenv import load_dotenv
load_dotenv()

ORG = os.getenv("AZURE_DEVOPS_ORG")
PAT = os.getenv("AZURE_DEVOPS_PAT")
if not ORG or not PAT:
        raise EnvironmentError("[FATAL] Missing env vars")

_auth = base64.b64encode(f":{PAT}".encode()).decode()
BASE_URL = f"https://dev.azure.com/{ORG}"
HEADERS = {"Authorization": f"Basic {_auth}", "Content-Type": "application/json"}

async def fetch_pr_details(project: str, repo_id: str, pr_id: int) -> dict:
        url = f"{BASE_URL}/{project}/_apis/git/repositories/{repo_id}/pullrequests/{pr_id}"
        async with httpx.AsyncClient() as client:
                r = await client.get(url, headers=HEADERS, params={"api-version": "7.1"})
                r.raise_for_status()
                return r.json()
```

### Tool Registration Pattern (both languages)

```typescript
// Node.js — always wrap in try/catch, return isError on failure
server.tool("get_pr_details", "description...", { schema }, async (args) => {
    try {
        const data = await fetchPRDetails(...);
        return { content: [{ type: "text", text: JSON.stringify(data, null, 2) }] };
    } catch (err: any) {
        return { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true };
    }
});
```

```python
# Python — always wrap in try/except, return error JSON
@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
        try:
                if name == "get_pr_details":
                        data = await fetch_pr_details(...)
                        return [TextContent(type="text", text=json.dumps(data, indent=2))]
        except Exception as e:
                return [TextContent(type="text", text=json.dumps({"error": str(e)}))]
```

---

## ✅ Testing with MCP Inspector UI

The **MCP Inspector** is a browser-based UI for interactively testing and debugging your MCP server tools — without needing VS Code or an LLM in the loop.

### Launch the Inspector

```bash
# Node.js (build first)
npm run build
npx @modelcontextprotocol/inspector node dist/index.js

# Python
npx @modelcontextprotocol/inspector python src/main.py
```

> The Inspector opens at **`http://localhost:5173`** by default.

### What You Can Do in the Inspector UI

| Feature | Description |
|---------|-------------|
| **List Tools** | See all registered tools with their names, descriptions, and input schemas |
| **Call a Tool** | Invoke any tool with custom JSON arguments and see the raw response |
| **Inspect Errors** | Surface `isError: true` responses and error messages immediately |
| **View Protocol Messages** | See the raw MCP JSON-RPC messages exchanged between client and server |

### Workflow: Inspector-First Development

1. Build/start your server
2. Open Inspector UI — verify all tools appear under the **Tools** tab
3. Call each tool manually with test arguments
4. Confirm correct response shape and `isError` handling
5. Only then wire into VS Code via `.vscode/mcp.json`

> **Tip:** Always test in the Inspector before connecting to an LLM — it's faster to catch schema mismatches and API errors here than mid-conversation.

---

## ✅ VS Code MCP Registration

```json
// .vscode/mcp.json
{
    "servers": {
        "azure-devops-pr-review": {
            "type": "stdio",
            "command": "node",
            "args": ["dist/index.js"],
            "env": {
                "AZURE_DEVOPS_ORG": "${env:AZURE_DEVOPS_ORG}",
                "AZURE_DEVOPS_PAT": "${env:AZURE_DEVOPS_PAT}"
            }
        }
    }
}
```

---

## ✅ Azure DevOps API Reference (APIs Used)

| What | Endpoint |
|------|----------|
| PR Details | `GET /{project}/_apis/git/repositories/{repo}/pullrequests/{prId}?api-version=7.1` |
| PR Iterations | `GET /{project}/_apis/git/repositories/{repo}/pullrequests/{prId}/iterations?api-version=7.1` |
| Changed Files | `GET /{project}/_apis/git/repositories/{repo}/pullrequests/{prId}/iterations/{iterId}/changes?api-version=7.1` |
| Post Comment | `POST /{project}/_apis/git/repositories/{repo}/pullrequests/{prId}/threads?api-version=7.1` |

---

## ✅ Design Guidelines / Best Practices

- **Validate env vars at startup** — don't let the server silently fail mid-request
- **All logging to stderr** — stdout is the MCP protocol channel, don't pollute it
- **One API client, shared** — don't create new HTTP clients per tool call
- **Return `isError: true`** on tool failure — LLM handles it gracefully vs a crash
- **Use `api-version: 7.1`** — always pin the Azure DevOps API version explicitly
- **Strip `refs/heads/`** from branch names before returning — cleaner for LLM
- **Test with MCP Inspector first** — validate tool schemas and responses before connecting to VS Code or an LLM

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| `console.log()` to stdout | Corrupts MCP protocol stream — use `console.error()` |
| PAT in source code | Security breach — always use `.env` |
| No `api-version` in requests | Azure DevOps may return unexpected schema versions |
| Creating HTTP client per request | Performance hit on every tool call |
| No try/catch in tools | One API failure crashes the entire server |
| Committing `.env` file | Exposes PAT — always in `.gitignore` |
| Skipping Inspector testing | Schema errors and bad responses only surface during LLM calls — harder to debug |

---

## ✅ How I Will Use This in My MCP Project

- My project will follow the exact structure above (separate `azure/`, `tools/`, `utils/`)
- PAT and org stored in `.env` — validated at startup with `process.exit(1)` / `raise` on missing
- All Azure DevOps calls go through a single shared client (`devopsClient` / `httpx.AsyncClient`)
- Every tool has `try/catch` returning `isError: true` — server never crashes on API errors
- All logging goes to `stderr` — stdout reserved for MCP protocol
- **Use MCP Inspector UI** to test every tool interactively before registering in VS Code
- Register server in `.vscode/mcp.json` with env var passthrough

---

*Next: Section 6 — Azure DevOps Use Case (Detailed PR review flow, all APIs, structuring for LLM)*