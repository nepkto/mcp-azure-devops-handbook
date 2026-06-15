# Section 3: MCP Server Design

---

## ✅ Summary

MCP Server design is about making your tools **LLM-readable and LLM-usable**. The LLM is smart but
relies entirely on your tool names, descriptions, and schemas to make decisions. Good design = the
LLM always picks the right tool, passes the right arguments, and gets useful output. Bad design =
wrong tool calls, wrong arguments, confused responses.

---

## ✅ Key Concepts

- **Tool Naming:** Use `verb_noun` format — action-oriented and unambiguous
- **Input Schema:** Every field needs a `description` — the LLM uses it to extract values from user input
- **Output Structure:** Return clean, filtered, structured JSON — not raw API responses
- **Single Responsibility:** One tool = one clear action
- **Boundary Setting:** Tell the LLM what a tool does NOT do (guides it to the right tool)
- **Type Correctness:** Use `number` for IDs, `string` for names, `enum` for fixed choices

---

## ✅ Tool Naming Rules

```
Pattern: [verb]_[noun]_[optional_qualifier]

✅ GOOD                         ❌ BAD
─────────────────────           ──────────────────
get_pr_details                  pr / do_stuff / tool1
get_pr_diff                     pr_data / fetch
post_review_comment             handle / process
list_pr_changed_files           get_info
detect_code_smells              analyze
summarize_pr_changes            stuff
```

---

## ✅ Code Snippets

### Node.js (TypeScript) — Good Tool Design

```typescript
server.tool(
  "get_pr_details",
  `Fetches metadata and status of a Pull Request in Azure DevOps.
   Use this when the user wants to know the title, description, author,
   status, reviewers, or target branch of a specific PR.
   Do NOT use this to get code changes — use get_pr_diff for that.`,
  {
    pr_id: z.number().describe(
      "The numeric Pull Request ID (e.g., 42). Found in the Azure DevOps PR URL."
    ),
    project: z.string().describe("The Azure DevOps project name (e.g., 'MyApp')"),
    repository: z.string().describe("The repository name within the project (e.g., 'backend-api')"),
  },
  async ({ pr_id, project, repository }) => {
    const pr = await fetchPRDetails(pr_id, project, repository);
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          pr_id: pr.pullRequestId,
          title: pr.title,
          status: pr.status,
          author: pr.createdBy.displayName,
          source_branch: pr.sourceRefName,
          target_branch: pr.targetRefName,
          reviewers: pr.reviewers.map(r => ({ name: r.displayName, vote: r.vote })),
          description: pr.description?.substring(0, 500)  // Truncate!
        }, null, 2)
      }]
    };
  }
);
```

### Python — Good Tool Design

```python
Tool(
    name="get_pr_details",
    description="""Fetches metadata and status of a Pull Request in Azure DevOps.
    Use this when the user wants to know the title, description, author,
    status, reviewers, or target branch of a specific PR.
    Do NOT use this to get code changes — use get_pr_diff for that.""",
    inputSchema={
        "type": "object",
        "properties": {
            "pr_id": {
                "type": "integer",
                "description": "The numeric Pull Request ID (e.g., 42)."
            },
            "project": {
                "type": "string",
                "description": "The Azure DevOps project name (e.g., 'MyApp')"
            },
            "repository": {
                "type": "string",
                "description": "The repository name within the project"
            }
        },
        "required": ["pr_id", "project", "repository"]
    }
)

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "get_pr_details":
        pr = await fetch_pr_details(arguments["pr_id"], arguments["project"], arguments["repository"])
        result = {
            "pr_id": pr["pullRequestId"],
            "title": pr["title"],
            "status": pr["status"],
            "author": pr["createdBy"]["displayName"],
            "reviewers": [{"name": r["displayName"], "vote": r["vote"]} for r in pr["reviewers"]],
            "description": pr.get("description", "")[:500]  # Truncate!
        }
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
```

---

## ✅ Good vs Bad Design Comparison

| Aspect         | ❌ Bad                              | ✅ Good                                      |
|----------------|-------------------------------------|----------------------------------------------|
| Tool name      | `pr`, `do_stuff`, `tool1`           | `get_pr_details`, `post_review_comment`      |
| Description    | "Gets PR info"                      | When to use + what it does + what it doesn't |
| Field types    | `id: string` for a PR ID           | `pr_id: number/integer`                      |
| Field describe | `pr_id: z.number()`                 | `pr_id: z.number().describe("The PR ID...")`  |
| Output         | Raw Azure DevOps API JSON           | Filtered, renamed, truncated, structured     |
| Scope          | One tool does get + post + analyze  | One tool, one action                         |
| Error handling | Throws unhandled exception          | Returns `{ error: "PR not found" }`          |

---

## ✅ Output Design Rules

```
DO:
  ✅ Return only the fields the LLM needs
  ✅ Use clear, readable field names (not Azure API jargon)
  ✅ Truncate long text (descriptions, commit messages) to ~500 chars
  ✅ Flatten nested objects where possible
  ✅ Add computed/derived fields (e.g., risk_level, change_summary)

DON'T:
  ❌ Return the raw Azure DevOps API response
  ❌ Return HTML, XML, or binary data
  ❌ Return full file contents when a summary will do
  ❌ Return deeply nested objects with Azure-internal field names
```

---

## ✅ Design Guidelines / Best Practices

- **"Do NOT use this for X — use Y instead"** → boundary setting in descriptions prevents misuse
- **Truncate long text** — descriptions, commit messages, file contents eat tokens fast
- **Add derived fields** — `risk_level`, `has_conflicts`, `is_approved` help the LLM reason faster
- **Use enums** for fixed values — `status: z.enum(["active", "completed", "abandoned"])`
- **Validate strictly** — reject bad input with a clear error message before touching Azure DevOps API
- **Keep output shape consistent** — same tool should always return same structure

---

## ✅ Common Mistakes

| Mistake                                  | Why It Fails                                              |
|------------------------------------------|-----------------------------------------------------------|
| Vague tool name (`pr`, `data`, `tool1`)  | LLM cannot infer when to use it                          |
| No descriptions on schema fields         | LLM guesses arguments — often wrong                      |
| Returning raw Azure DevOps API JSON      | Too much noise, Azure-internal naming confuses the LLM   |
| One mega-tool doing everything           | LLM doesn't know when to use it; hard to maintain        |
| Wrong field types (string for IDs)       | LLM passes `"42"` instead of `42` → API call fails       |
| No boundary hints in description         | LLM uses wrong tool when similar tools exist             |
| Forgetting to truncate large text fields | Token budget blown, LLM loses context                    |

---

## ✅ Tool Design Checklist

```
Before registering any tool, verify:

[ ] Name follows verb_noun pattern and is self-explanatory
[ ] Description explains WHEN to use it (not just what it does)
[ ] Description says what it does NOT do (if similar tools exist)
[ ] Every input field has .describe() / description
[ ] Required vs optional fields are correctly marked
[ ] Input types are correct (number for IDs, not string)
[ ] Output is structured, filtered JSON (not raw API response)
[ ] Long text fields are truncated (~500 chars max)
[ ] Errors are caught and returned as meaningful messages
[ ] Tool does ONE thing only
```

---

## ✅ Azure DevOps PR Example

**3 well-designed tools for PR review:**

| Tool Name              | When LLM Calls It                            | What It Returns                        |
|------------------------|----------------------------------------------|----------------------------------------|
| `get_pr_details`       | "Tell me about PR #42"                        | Title, author, status, reviewers       |
| `get_pr_diff`          | "Review the code changes in PR #42"           | Changed files, diff summaries          |
| `post_review_comment`  | "Post a comment about the SQL injection risk" | Confirmation of posted comment         |

Each tool is **focused, well-named, and independently useful** — the LLM can compose them naturally.

---

## ✅ How I Will Use This in My MCP Project

- Every tool I build will follow the `verb_noun` naming pattern
- All tool descriptions will answer: **When to use it** + **What it does** + **What it does NOT do**
- Every schema field will have `.describe()` (Node.js) or `"description"` (Python)
- Output will always be **filtered, structured JSON** — never raw Azure DevOps API responses
- I will add derived fields like `risk_level` and `has_security_issues` to help the LLM reason faster
- I will add a **truncation layer** for large fields (PR descriptions, diffs, comments)

---

*Next: Section 4 — Data Flow & Context Handling (Chunking, token limits, handling large PR diffs)*