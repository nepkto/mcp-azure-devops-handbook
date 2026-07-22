# Section 8: Advanced Patterns

---

## ✅ Summary

Tool composition moves multi-step orchestration from the LLM into the server itself, by building
higher-level tools (`review_pr`, `detect_code_smells`) that internally call existing primitive
tools (`get_pr_details`, `get_pr_diff`, `get_pr_comments`). This reduces LLM round-trips, enforces
call order server-side, and creates a seam for adding organization-specific business rules on top
of generic risk detection.

---

## ✅ Key Concepts

- **Tool composition** = a new tool that calls other tools' `handle()` functions internally and
  combines their output, instead of making the LLM orchestrate multiple separate calls
- **Composed tools trade flexibility for consistency** — keep both primitives AND composed tools;
  let boundary-setting descriptions guide the LLM to the cheaper option when appropriate
- **Domain-specific logic layer** — functions like `_score_severity()` are where your
  organization's actual standards live, separate from generic pattern detection
- **Reuse, don't reimplement** — composed tools call existing `handle()` functions rather than
  duplicating Azure DevOps API logic
- **Layer responsibility stays clean**: `devops_client.py` = pure API wrapper, `utils/summarize.py`
  = generic risk detection, `tools/detect_code_smells.py` = business rules on top

---

## ✅ Composition Pattern

```
WITHOUT composition:                          WITH composition:
LLM orchestrates 4 separate calls        →    LLM makes 1 call: review_pr(42)
  1. get_pr_details(42)                          │
  2. get_pr_diff(42)                             ▼
  3. get_pr_comments(42)                    Internally calls all 3 read primitives
  4. post_review_comment(42, "...")         concurrently, combines into one payload
```

---

## ✅ Code Snippets

### Python — `review_pr` (Composed Tool)

```python
from src.tools.get_pr_details import handle as handle_get_pr_details
from src.tools.get_pr_diff import handle as handle_get_pr_diff
from src.tools.get_pr_comments import handle as handle_get_pr_comments

async def handle(arguments: dict) -> list[TextContent]:
    pr_id, project, repository = arguments["pr_id"], arguments["project"], arguments["repository"]
    try:
        details_result, diff_result, comments_result = await asyncio.gather(
            handle_get_pr_details({"pr_id": pr_id, "project": project, "repository": repository}),
            handle_get_pr_diff({"pr_id": pr_id, "project": project, "repository": repository}),
            handle_get_pr_comments({"pr_id": pr_id, "project": project, "repository": repository}),
        )
        combined = {
            "pr_details": json.loads(details_result[0].text),
            "pr_diff":    json.loads(diff_result[0].text),
            "pr_comments": json.loads(comments_result[0].text),
        }
        return [TextContent(type="text", text=json.dumps(combined, indent=2))]
    except Exception as e:
        print(f"[ERROR] review_pr: {e}", file=sys.stderr)
        return [TextContent(type="text", text=json.dumps({"error": str(e), "pr_id": pr_id}))]
```

### Python — `detect_code_smells` (Composed + Domain Logic)

```python
SECURITY_KEYWORDS = {"SQL injection risk", "Hardcoded secret", "eval() usage",
                      "Shell injection risk", "Auth bypass pattern", "Exposed debug endpoint"}
MAINTAINABILITY_KEYWORDS = {"TODO / FIXME left in", "print() left in",
                             "Commented-out code", "Broad exception caught"}

def _score_severity(security_count: int, maintainability_count: int) -> str:
    if security_count >= 1:
        return "high"
    if maintainability_count >= 2:
        return "medium"
    return "low"

async def handle(arguments: dict) -> list[TextContent]:
    pr_id, project, repository = arguments["pr_id"], arguments["project"], arguments["repository"]
    try:
        diff_result = await handle_get_pr_diff({"pr_id": pr_id, "project": project, "repository": repository})
        diff_data = json.loads(diff_result[0].text)

        smells_by_file = []
        for f in diff_data.get("files", []):
            signals = f.get("risk_signals", [])
            security = [s for s in signals if s in SECURITY_KEYWORDS]
            maintain = [s for s in signals if s in MAINTAINABILITY_KEYWORDS]
            if not security and not maintain:
                continue
            smells_by_file.append({
                "file": f["file"], "severity": _score_severity(len(security), len(maintain)),
                "security_smells": security, "maintainability_smells": maintain,
            })

        rank = {"high": 0, "medium": 1, "low": 2}
        smells_by_file.sort(key=lambda f: rank[f["severity"]])

        result = {
            "pr_id": pr_id, "files_with_smells": len(smells_by_file),
            "high_severity_count": sum(1 for f in smells_by_file if f["severity"] == "high"),
            "smells": smells_by_file,
        }
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    except Exception as e:
        print(f"[ERROR] detect_code_smells: {e}", file=sys.stderr)
        return [TextContent(type="text", text=json.dumps({"error": str(e), "pr_id": pr_id}))]
```

### Domain-Specific Rule Example

```python
# Enterprise rule layered on top of generic severity scoring
def _score_severity(security_count: int, maintainability_count: int, file_path: str) -> str:
    if any(critical in file_path.lower() for critical in ["payment", "auth", "billing"]):
        if security_count >= 1 or maintainability_count >= 1:
            return "critical"   # Custom tier beyond high/medium/low
    if security_count >= 1:
        return "high"
    if maintainability_count >= 2:
        return "medium"
    return "low"
```

---

## ✅ Design Guidelines / Best Practices

- **Compose by calling existing `handle()` functions** — never duplicate Azure DevOps API logic
  inside a composed tool
- **Keep primitives available alongside composed tools** — narrow questions should still use the
  cheap, specific tool
- **Use `asyncio.gather` / `Promise.all`** for independent primitive calls inside a composed tool —
  they don't depend on each other's results, so run them concurrently
- **Put business rules in the tool layer**, not in `devops_client.py` (pure API wrapper) or
  `utils/summarize.py` (generic pattern detection)
- **Sort composed/analyzed results by severity** — helps the LLM address the most important items
  first without re-deriving priority itself
- **Skip "clean" results in domain-logic tools** — don't waste LLM tokens listing files with zero
  smells

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| Reimplementing Azure DevOps calls inside a composed tool | Duplicate logic, drifts out of sync with primitives |
| Removing primitive tools once a composed tool exists | Forces expensive full-review calls even for narrow questions |
| Putting business rules inside `devops_client.py` | Breaks single-responsibility; API wrapper should stay generic |
| Sequential `await` calls for independent primitives | Unnecessary latency — use concurrent gather/Promise.all |
| Not describing when to use composed vs primitive tools | LLM can't tell which is cheaper/appropriate for the question |

---

## ✅ Azure DevOps PR Example

**User:** *"Give me a full review of PR #42"*
→ Composed tool `review_pr(42)` called once → internally fetches details + diff + comments
concurrently → LLM receives one combined payload → produces a complete review in a single
round-trip.

**User:** *"Just tell me who authored PR #42"*
→ Primitive tool `get_pr_details(42)` called directly → cheaper, faster, no wasted diff/comment
fetches.

---

## ✅ How I Will Use This in My MCP Project

- Add `review_pr` as a composed tool that calls the 3 existing read-tool `handle()` functions
  concurrently via `asyncio.gather`
- Add `detect_code_smells` as a domain-logic layer on top of `get_pr_diff`, categorizing
  `risk_signals` into security vs maintainability smells with a severity score
- Keep all 4 existing primitives unchanged and available for narrow questions
- Reserve `_score_severity()` (and similar functions) as the place to add organization-specific
  escalation rules (e.g., payment/auth files always escalate)
- Write descriptions for composed tools that explicitly tell the LLM when to prefer primitives
  instead

---

*Next: Section 9 — Performance & Scaling (large PRs, caching, rate limiting, efficient API calls)*