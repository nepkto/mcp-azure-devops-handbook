# Section 9: MCP Client Registration & Environment Configuration

---

## ✅ Summary

MCP Clients (VS Code, etc.) don't auto-discover servers — they launch them as subprocesses based
on a config file (`mcp.json`) that specifies the command, arguments, and environment. Two common
misconfigurations — a vague interpreter path and `${env:VAR}` substitution shadowing real secrets
from a `.env` file — can make a correctly-coded server fail or silently misbehave, in ways that
look like unrelated bugs (API 401s, "module not found" errors).

---

## ✅ Key Concepts

- **`mcp.json` is wiring, not logic** — it tells the client which interpreter to spawn, with what
  arguments, and what environment; it contains no behavior of its own
- **Interpreter resolution via PATH is fragile** — `"command": "python"` depends entirely on
  what's first on the OS PATH at the moment VS Code spawns the process, which may not be your venv
- **`${env:VAR}` reads the OS environment, not your `.env` file** — if the OS var isn't set, VS
  Code injects an **empty string**, not nothing
- **`python-dotenv`'s `load_dotenv()` does not override existing env vars by default** — an empty
  string still counts as "existing", so your `.env` file's real value gets silently shadowed
- **Fail-fast checks turn silent bugs into loud ones** — the `if not PAT: sys.exit(1)` check in
  `devops_client.py` is what makes this bug visible instead of causing confusing downstream 401s

---

## ✅ The Bug Timeline

```
1. VS Code reads mcp.json → ${env:AZURE_DEVOPS_PAT} → not set on OS → resolves to ""
2. VS Code spawns: python main.py  with AZURE_DEVOPS_PAT="" in the environment
3. main.py starts → devops_client.py calls load_dotenv()
4. load_dotenv() sees AZURE_DEVOPS_PAT already exists (as "") → does NOT load .env's real value
5. Fail-fast check triggers: "AZURE_DEVOPS_PAT must be set" → crashes
   (or without the check: silent empty-string auth failures downstream)
```

---

## ✅ Code Snippets

### ❌ Incorrect `mcp.json` (Python server)

```json
{
  "servers": {
    "azure-devops-pr-review": {
      "type": "stdio",
      "command": "python",
      "args": ["main.py"],
      "env": {
        "AZURE_DEVOPS_ORG": "${env:AZURE_DEVOPS_ORG}",
        "AZURE_DEVOPS_PAT": "${env:AZURE_DEVOPS_PAT}"
      }
    }
  }
}
```

### ✅ Correct `mcp.json` (Python server)

```json
{
  "servers": {
    "azure-devops-pr-review": {
      "type": "stdio",
      "command": "${workspaceFolder}/.venv/Scripts/python.exe",
      "args": ["main.py"]
    }
  }
}
```

### ❌ Incorrect `mcp.json` (Node.js server)

```json
{
  "servers": {
    "azure-devops-pr-review": {
      "type": "stdio",
      "command": "node",
      "args": ["dist/index.js"],
      "env": { "AZURE_DEVOPS_PAT": "${env:AZURE_DEVOPS_PAT}" }
    }
  }
}
```

### ✅ Correct `mcp.json` (Node.js server)

```json
{
  "servers": {
    "azure-devops-pr-review": {
      "type": "stdio",
      "command": "node",
      "args": ["${workspaceFolder}/dist/index.js"]
    }
  }
}
```

---

## ✅ Design Guidelines / Best Practices

- **Use an absolute/explicit interpreter path** (`${workspaceFolder}/.venv/Scripts/python.exe`),
  never a bare command name that depends on PATH ordering
- **Let your server load its own `.env` file** via `load_dotenv()` — don't duplicate secret
  injection through the client's `env` block unless the server has no other way to get secrets
- **Only use `env` blocks in `mcp.json` when the server truly has no local secret-loading
  mechanism** (e.g., compiled binaries, CI/CD-injected secrets) — and confirm those OS vars are
  actually set before relying on `${env:VAR}`
- **Keep fail-fast validation in the server** (`if not PAT: sys.exit(1)`) regardless of how
  secrets are supplied — it converts silent misconfiguration into a loud, debuggable error
- **Verify registration by testing a real tool call**, not just by checking the server shows
  "connected" — a connected-but-misconfigured server still fails on first real use

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| `"command": "python"` with no path | Resolves to wrong/missing interpreter depending on system PATH |
| Assuming `${env:VAR}` reads `.env` | It reads the OS environment only — `.env` files are invisible to it |
| Adding an `env` block "just in case" | Can silently shadow real `.env` values with empty strings |
| Treating a 401 error as an Azure DevOps problem | The real cause may be the server never got valid secrets at startup |
| Skipping fail-fast validation to "keep code clean" | Turns a loud, debuggable crash into confusing downstream API errors |

---

## ✅ Azure DevOps PR Example

**Symptom:** User asks Copilot "Get details for PR #1" and gets back
`{"error": "Client error '401 Unauthorized'..."}`.

**Wrong diagnosis:** "My PAT must be expired or wrong."

**Right diagnosis (in this project's case):** `mcp.json`'s `env` block injected
`AZURE_DEVOPS_PAT=""` because the OS environment variable was never set, which blocked
`load_dotenv()` from loading the real (valid) PAT from `.env`. Removing the `env` block and
pointing `command` directly at the venv Python fixed it — the PAT was fine all along.

---

## ✅ How I Will Use This in My MCP Project

- Set `"command"` in `.vscode/mcp.json` to the explicit venv interpreter path, never a bare name
- Remove the `env` block from `mcp.json` since `devops_client.py` already handles secrets via
  `load_dotenv()` — avoid the double-source-of-truth risk
- Keep the existing fail-fast validation in `devops_client.py` as a safety net regardless of how
  the server is launched
- When debugging a tool-call failure, check **server startup/registration first** before assuming
  the bug is in Azure DevOps API logic

---

*Next: Section 10 — Performance & Scaling (large PRs, caching, rate limiting, efficient API calls)*