# Section 6: Azure DevOps Use Case — PR Review

---

## ✅ Summary

PR review is a sequence of small information-gathering steps followed by a judgment call — not
one atomic action. MCP tools map 1:1 onto that human workflow: fetch metadata, fetch the diff,
check existing comments, then post feedback. This section catalogs every Azure DevOps API needed
for PR review and shows the ideal LLM-facing payload shape that ties `get_pr_details`,
`get_pr_diff`, and `get_pr_comments` together.

---

## ✅ Key Concepts

- **PR review = 4 human questions** → "What is this PR?" → "What changed?" → "What's already been
  said?" → "What feedback should I give?" — each maps to one tool
- **Azure DevOps has no single "diff" endpoint** — `/iterations/{id}/changes` gives file paths
  only; real diff content requires fetching file blobs at both commits and diffing locally
  (see [devops_client.py](../python-azure-devops-mcp/src/azure/devops_client.py))
- **Read tools vs write tools** — `get_pr_details`/`get_pr_diff`/`get_pr_comments` are safe to call
  freely; `post_review_comment` is the only tool that mutates Azure DevOps state
- **Checking existing comments prevents duplicate feedback** — critical for a good AI reviewer UX
- **Final LLM payload is assembled from 3 tool calls**, each independently filtered/summarized

---

## ✅ Required Azure DevOps APIs

| Need | API Endpoint | Used By |
|------|-------------|---------|
| PR metadata | `GET /{project}/_apis/git/repositories/{repo}/pullrequests/{prId}` | `get_pr_details` |
| PR iterations (pushes) | `GET .../pullrequests/{prId}/iterations` | `get_pr_diff` (finds latest push) |
| Changed files list (no diff content) | `GET .../iterations/{iterId}/changes` | `get_pr_diff` (path list only) |
| Commit diff (real content) | `GET .../repositories/{repo}/diffs/commits` | `get_pr_diff` (primary source of file list) |
| Raw file content at commit | `GET .../repositories/{repo}/items?versionDescriptor...` | `get_pr_diff` (before/after blobs) |
| PR comment threads | `GET .../pullrequests/{prId}/threads` | `get_pr_comments` |
| Post a comment | `POST .../pullrequests/{prId}/threads` | `post_review_comment` |

All endpoints pinned to `api-version=7.1` (see Section 5 best practices).

---

## ✅ Full PR Review Data Flow

```
get_pr_details(pr_id)
   → title, author, status, branches, reviewers
        │
        ▼
get_pr_diff(pr_id)
   a. fetch_pr_details() → source_commit + target_commit
   b. /diffs/commits → changed file list between commits
   c. fetch file content @ both commits, concurrently, per file
   d. difflib.unified_diff() → real diff string
   e. filter_changed_files() → drop lock/binary/generated files
   f. summarize_all_files() → risk_signals + lines_added/removed
        │
        ▼
get_pr_comments(pr_id)
   → existing threads, so LLM avoids repeating feedback
        │
        ▼
LLM reasons over all 3 results → drafts review comments
        │
        ▼
post_review_comment(pr_id, comment)
   → posted as a PR thread, visible in Azure DevOps UI
```

---

## ✅ Code Snippets

### Node.js — `get_pr_comments` Tool

```typescript
server.tool(
  "get_pr_comments",
  `Fetches all existing review comment threads on a Pull Request in Azure DevOps.
   Use this BEFORE posting new feedback, to avoid duplicate comments.
   Returns thread status (active/resolved) and comment authors.`,
  {
    pr_id: z.number().describe("The numeric Pull Request ID"),
    project: z.string().describe("Azure DevOps project name"),
    repository: z.string().describe("Repository name"),
  },
  async ({ pr_id, project, repository }) => {
    try {
      const threads = await fetchPRComments(project, repository, pr_id);
      const humanThreads = threads.filter((t: any) =>
        t.comments?.some((c: any) => c.commentType === 1)
      );
      const summary = humanThreads.map((t: any) => ({
        thread_id: t.id,
        status: t.status,
        file: t.threadContext?.filePath ?? null,
        comments: t.comments
          .filter((c: any) => c.commentType === 1)
          .map((c: any) => ({ author: c.author.displayName, content: c.content?.substring(0, 300) })),
      }));
      return {
        content: [{ type: "text", text: JSON.stringify({
          pr_id,
          total_threads: summary.length,
          unresolved_threads: summary.filter((t: any) => t.status === "active").length,
          threads: summary,
        }, null, 2) }],
      };
    } catch (err: any) {
      return { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true };
    }
  }
);
```

### Python — `get_pr_comments` Tool

```python
async def handle(arguments: dict) -> list[TextContent]:
    pr_id, project, repository = arguments["pr_id"], arguments["project"], arguments["repository"]
    try:
        threads = await fetch_pr_comments(project, repository, pr_id)
        summary = []
        for t in threads:
            human_comments = [c for c in t.get("comments", []) if c.get("commentType") == 1]
            if not human_comments:
                continue
            summary.append({
                "thread_id": t["id"],
                "status": t.get("status", "unknown"),
                "file": t.get("threadContext", {}).get("filePath"),
                "comments": [
                    {"author": c["author"]["displayName"], "content": c.get("content", "")[:300]}
                    for c in human_comments
                ],
            })
        result = {
            "pr_id": pr_id,
            "total_threads": len(summary),
            "unresolved_threads": sum(1 for t in summary if t["status"] == "active"),
            "threads": summary,
        }
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    except Exception as e:
        print(f"[ERROR] get_pr_comments: {e}", file=sys.stderr)
        return [TextContent(type="text", text=json.dumps({"error": str(e), "pr_id": pr_id}))]
```

---

## ✅ Final LLM-Facing Payload Shape

```json
{
  "pr_details": {
    "pr_id": 42, "title": "Add user search endpoint", "author": "Alice",
    "status": "active", "source_branch": "feature/user-search", "target_branch": "main"
  },
  "pr_diff": {
    "total_files": 12, "reviewed_files": 5, "skipped_files": 7, "high_risk_count": 2,
    "files": [
      { "file": "src/api/userController.py", "lines_added": 34, "lines_removed": 3,
        "risk_signals": ["SQL injection risk"], "has_high_risk": true,
        "preview": "+ cursor.execute(\"SELECT * FROM users WHERE id=\" + user_id)..." }
    ]
  },
  "pr_comments": {
    "total_threads": 1, "unresolved_threads": 1,
    "threads": [
      { "thread_id": 3, "status": "active", "file": "src/api/userController.py",
        "comments": [{ "author": "Bob", "content": "Can we add pagination here?" }] }
    ]
  }
}
```

---

## ✅ Design Guidelines / Best Practices

- **Always call `get_pr_comments` before `post_review_comment`** — avoid the LLM repeating
  feedback that's already been given
- **Never fetch diff content via `/iterations/changes` alone** — it has no diff data; combine with
  `/diffs/commits` + raw file fetch, as `fetch_pr_changes_with_diff()` does
- **Filter out system-generated comment threads** (`commentType !== 1`) — Azure DevOps injects
  automated status comments that aren't useful PR feedback
- **Keep read tools and write tools clearly separated** — makes it trivial to add permission
  checks or a confirmation step before anything is posted
- **Every payload section should be independently requestable** — don't force the LLM to fetch
  everything when it only needs PR metadata

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| Assuming `/iterations/changes` includes diff content | It never does — Azure DevOps limitation |
| Posting comments without checking existing threads first | Duplicate/redundant feedback, poor UX |
| Including system-generated comments in `get_pr_comments` | Clutters output with noise, not real feedback |
| Returning nested Azure-internal JSON straight to the LLM | Wastes tokens, confuses the model |
| Treating PR review as one big tool call | Loses the ability to answer partial questions cheaply |

---

## ✅ Azure DevOps PR Example

**User:** *"Review PR #42 and check if anyone already flagged issues before you comment"*

1. LLM calls `get_pr_details(42)` → learns it's "Add user search endpoint" by Alice
2. LLM calls `get_pr_diff(42)` → finds `userController.py` has a SQL injection risk signal
3. LLM calls `get_pr_comments(42)` → sees Bob already asked about pagination (unrelated to SQL risk)
4. LLM drafts a comment specifically about the SQL injection risk (not duplicating Bob's comment)
5. LLM calls `post_review_comment(42, "...")` → comment appears in Azure DevOps PR timeline

---

## ✅ How I Will Use This in My MCP Project

- Add the missing `get_pr_comments` tool to `python-azure-devops-mcp/src/tools/` following the
  exact pattern of the other 3 tools (`TOOL_DEFINITION` + `async def handle`)
- Wire it into `main.py` with a `@mcp.tool()` wrapper, matching `project`/`repository` defaults
- Add `fetch_pr_comments()` to `devops_client.py` (already scaffolded in Section 5)
- Filter out `commentType != 1` (system comments) before returning to the LLM
- Document the full 3-tool payload shape as the "contract" my MCP server offers for PR review

---

*Next: Section 7 — AI + MCP Interaction (prompt design, tool selection, high-quality PR reviews)*
