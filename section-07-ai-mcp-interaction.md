# Section 7: AI + MCP Interaction

---

## ✅ Summary

Copilot never sees your server's code — only the tool manifest (`name`, `description`,
`inputSchema`) for every registered tool. Tool selection is a joint outcome of the user's prompt
and your tool descriptions. Weak prompts or weak descriptions cause wrong or excessive tool calls;
strong versions of both make reviews reliable, scoped, and efficient.

---

## ✅ Key Concepts

- **Tool manifest is Copilot's only window into your server** — no code, no comments, just
  `name` + `description` + `inputSchema`
- **Tool selection = prompt intent ∩ description purpose** — Copilot matches user language against
  description language
- **Vague prompts cause extra tool calls** — Copilot over-fetches to compensate for ambiguity,
  costing tokens and time
- **Good descriptions compensate for bad prompts, but not vice versa** — invest in descriptions
  first, prompts second
- **Boundary setting scales with tool count** — the more tools you have, the more critical
  "Do NOT use this for X — use Y" becomes to avoid overlap confusion

---

## ✅ The Bidirectional Relationship

```
Bad prompt  + Bad description  → Wrong tool / no tool called
Bad prompt  + Good description → Copilot still infers correctly
Good prompt + Bad description  → Wrong tool called anyway
Good prompt + Good description → Reliable, efficient tool use
```

---

## ✅ Prompt → Tool Mapping (This Project)

| User Prompt | Matched Tool | Why |
|-------------|-------------|-----|
| "Who created PR #42?" | `get_pr_details` | description mentions "author" |
| "Review the code changes in #42" | `get_pr_diff` | description mentions "review code changes" |
| "Has anyone already commented?" | `get_pr_comments` | description mentions "existing feedback" |
| "Post a comment about the SQL risk" | `post_review_comment` | explicit "post"/"submit" verb match |
| "Review PR #42" (vague) | Likely `get_pr_details` + `get_pr_diff` | Copilot gathers broadly when scope is unclear |

---

## ✅ Practical Prompt Templates

| Goal | Prompt Template |
|------|-----------------|
| Security-focused review | "Review PR #{id} for security issues only. Focus on files where `has_high_risk` is true. Explain each risk and suggest a fix." |
| Avoid duplicate feedback | "Before reviewing PR #{id}, check get_pr_comments. Only raise points not already discussed." |
| Post directly | "Review PR #{id} for SQL injection and hardcoded secrets. If you find any, post a comment summarizing the issues." |
| Scoped file review | "Review only the changes to `userController.py` in PR #{id}. Ignore other files." |
| Comparative review | "Compare PR #{id}'s reviewer votes with the actual risk signals — flag if approved despite unresolved high-risk files." |

---

## ✅ Design Guidelines / Best Practices

- **Write descriptions assuming zero code context** — Copilot only reads the manifest, never the
  implementation
- **Reference real output field names in prompts** (e.g. `has_high_risk`, `risk_signals`) — this
  primes Copilot to prioritize exactly those fields when reasoning
- **Enforce workflow order via prompt, not just code** — e.g. explicitly say "check comments
  first" rather than assuming Copilot will always call tools in the ideal order
- **Bound output length/scope explicitly** — "under 150 words per file", "security issues only" —
  prevents generic, unfocused reviews
- **Keep every new tool's description mutually exclusive from existing ones** — always add a
  "Do NOT use this for X — use Y" clause once you have 2+ similar tools

---

## ✅ Common Mistakes

| Mistake | Why It Fails |
|--------|-------------|
| Assuming Copilot infers intent from vague prompts | It over-fetches or picks the wrong tool |
| Writing descriptions "for humans reading the code" | Copilot never sees code — only the manifest |
| Adding new tools without checking overlap with existing descriptions | Ambiguous tool selection at scale |
| Not referencing actual data field names in prompts | Copilot may miss which fields matter most |
| Expecting consistent output without bounding scope/length in the prompt | Generic, inconsistent reviews |

---

## ✅ Azure DevOps PR Example

**Weak:** `"Review PR #42"` → generic review, inconsistent depth, possibly redundant feedback

**Strong:**
```
Review PR #42 for security issues only. For each file flagged with a risk_signal
in get_pr_diff, explain the specific risk in plain language and suggest a concrete
fix. Check get_pr_comments first — don't repeat feedback that's already there.
Keep the review under 150 words per file.
```
→ Copilot calls `get_pr_comments` → `get_pr_diff` → filters to `has_high_risk` files → produces a
scoped, non-redundant, security-focused review.

---

## ✅ How I Will Use This in My MCP Project

- Keep every tool description written as if Copilot has **never seen my code** — plain language,
  explicit "when to use" and "do NOT use for"
- Provide prompt templates (like the table above) to end users of my MCP server as a "how to get
  good reviews" cheat sheet
- When adding new tools beyond the current 4, audit all descriptions together for overlap before
  shipping
- Reference actual JSON field names (`risk_signals`, `has_high_risk`, `unresolved_threads`) in
  example prompts so users learn to write precise requests

---

*Next: Section 8 — Advanced Patterns (tool composition, review_pr, detect_code_smells, domain rules)*
