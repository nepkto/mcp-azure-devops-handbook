# Section 4: Data Flow & Context Handling

---

## ✅ Summary

Every piece of data sent through MCP consumes LLM tokens. Large PR diffs (50+ files, 10,000+ lines)
can blow the token budget, causing the LLM to lose context and produce poor reviews. The solution is
a **3-stage data pipeline**: Filter → Chunk → Summarize — applied *before* data reaches the LLM.

---

## ✅ Key Concepts

- **Token Budget:** LLM has a fixed context window (e.g., 128k tokens) shared by system prompt,
  chat history, tool schemas, AND tool results
- **Filter:** Remove noise — binary files, lock files, generated files, snapshots
- **Chunk:** Split large diffs into fixed-size pieces (e.g., 400 lines max) for progressive review
- **Summarize:** Pre-process diffs into risk signals (`has_sql_risk`, `has_secrets_risk`) so the
  LLM reasons faster with less data
- **Token waste = quality loss** — the more tokens spent on noise, the less the LLM can reason

---

## ✅ The 3-Stage Pipeline

```
Azure DevOps API (Raw diff: 10,000 lines, 50 files)
    │
    ▼
STAGE 1: FILTER
    Remove: binary files, lock files, dist/, auto-generated files
    Result: ~20 files, ~3,000 lines
    │
    ▼
STAGE 2: CHUNK
    Split: max 400 lines per chunk per file
    Result: multiple focused, reviewable chunks
    │
    ▼
STAGE 3: SUMMARIZE
    Extract: lines_added, lines_removed, risk_signals, preview
    Result: compact, LLM-ready summaries with risk signals
    │
    ▼
LLM receives clean data → produces high-quality review
```

---

## ✅ Code Snippets

### Node.js — Filter

```typescript
const SKIP_PATTERNS = [
  /\.min\.(js|css)$/, /package-lock\.json$/, /yarn\.lock$/,
  /\.snap$/, /dist\//, /node_modules\//, /\.(png|jpg|svg)$/,
];

function filterChangedFiles(files: ChangedFile[]): ChangedFile[] {
  return files.filter(file =>
    !SKIP_PATTERNS.some(pattern => pattern.test(file.path))
  );
}
```

### Python — Filter

```python
SKIP_PATTERNS = [r"\.min\.(js|css)$", r"package-lock\.json$", r"\.snap$", r"dist/"]

def filter_changed_files(files: list[dict]) -> list[dict]:
    return [f for f in files if not any(
        re.search(p, f.get("path",""), re.IGNORECASE) for p in SKIP_PATTERNS
    )]
```

### Node.js — Chunk

```typescript
const MAX_LINES = 400;

function chunkFileDiff(filePath: string, diffLines: string[]): DiffChunk[] {
  return Array.from({ length: Math.ceil(diffLines.length / MAX_LINES) }, (_, i) => ({
    file: filePath,
    chunk_index: i + 1,
    total_chunks: Math.ceil(diffLines.length / MAX_LINES),
    lines: diffLines.slice(i * MAX_LINES, (i + 1) * MAX_LINES),
  }));
}
```

### Python — Chunk

```python
MAX_LINES = 400

def chunk_file_diff(file_path: str, diff_lines: list[str]) -> list[dict]:
    total_chunks = -(-len(diff_lines) // MAX_LINES)
    return [
        {"file": file_path, "chunk_index": i+1, "total_chunks": total_chunks,
         "lines": diff_lines[i*MAX_LINES:(i+1)*MAX_LINES]}
        for i in range(total_chunks)
    ]
```

### Node.js — Summarize (Risk Signals)

```typescript
const RISK_PATTERNS: Record<string, RegExp> = {
  "SQL injection risk":   /\+.*(query|execute|sql)\s*\(.*?\+/i,
  "Hardcoded secret":     /\+.*(password|secret|token|api_key)\s*=\s*["'][^"']+["']/i,
  "eval() usage":         /\+.*eval\s*\(/i,
  "TODO left in":         /\+.*(TODO|FIXME|HACK)/i,
  "Auth bypass pattern":  /\+.*(bypass|skip.*auth|disable.*auth)/i,
};

function summarizeFileDiff(file: string, diff: string): FileSummary {
  const lines = diff.split("\n");
  const risk_signals = Object.entries(RISK_PATTERNS)
    .filter(([, p]) => p.test(diff)).map(([label]) => label);
  return {
    file,
    lines_added: lines.filter(l => l.startsWith("+") && !l.startsWith("+++")).length,
    lines_removed: lines.filter(l => l.startsWith("-") && !l.startsWith("---")).length,
    risk_signals,
    has_secrets_risk: risk_signals.some(r => r.includes("secret")),
    has_sql_risk: risk_signals.some(r => r.includes("SQL")),
    preview: diff.substring(0, 300),
  };
}
```

### Python — Summarize (Risk Signals)

```python
RISK_PATTERNS = {
    "SQL injection risk":   r"\+.*(query|execute|sql)\s*\(.*?\+",
    "Hardcoded secret":     r"\+.*(password|secret|token|api_key)\s*=\s*[\"'][^\"']+[\"']",
    "eval() usage":         r"\+.*eval\s*\(",
    "TODO left in":         r"\+.*(TODO|FIXME|HACK)",
    "Auth bypass pattern":  r"\+.*(bypass|skip.*auth|disable.*auth)",
}

def summarize_file_diff(file_path: str, diff: str) -> dict:
    lines = diff.split("\n")
    risk_signals = [label for label, p in RISK_PATTERNS.items()
                    if re.search(p, diff, re.IGNORECASE)]
    return {
        "file": file_path,
        "lines_added": sum(1 for l in lines if l.startswith("+") and not l.startswith("+++")),
        "lines_removed": sum(1 for l in lines if l.startswith("-") and not l.startswith("---")),
        "risk_signals": risk_signals,
        "has_secrets_risk": any("secret" in r.lower() for r in risk_signals),
        "has_sql_risk": any("sql" in r.lower() for r in risk_signals),
        "preview": diff[:300]
    }
```

---

## ✅ Token Budget Reality Check

| Scenario                        | Approx Tokens | LLM Quality       |
|---------------------------------|---------------|-------------------|
| Raw 50-file PR diff             | ~80,000       | ❌ Barely fits    |
| After filtering (20 files)      | ~32,000       | ✅ Acceptable     |
| After chunking (400 lines/file) | ~6,000        | ✅✅ Good         |
| After summarizing (risk signals)| ~2,000        | ✅✅✅ Excellent  |

---

## ✅ Design Guidelines / Best Practices

- **Always filter before sending** — skip binary, generated, and dependency files
- **Set a hard chunk size** — 300–500 lines per chunk is a safe range
- **Pre-compute risk signals** — don't let the LLM scan 3,000 lines for a hardcoded password
- **Truncate previews** — 200–400 chars is enough context per file
- **Add metadata counts** — `total_files`, `reviewed_files`, `skipped_files` help LLM understand scope
- **Design chunked tools** — `get_pr_diff` (overview) + `get_pr_diff_chunk` (detail) = good pattern

---

## ✅ Common Mistakes

| Mistake                              | Why It Fails                                         |
|--------------------------------------|------------------------------------------------------|
| Sending raw git diff to LLM          | Noise overwhelms useful signal; wastes tokens        |
| No file filtering                    | Lock files, snapshots, minified files fill the budget|
| Single tool for all 50 files at once | Hits token limit; LLM loses early context            |
| No chunk index / total_chunks info   | LLM doesn't know how to request next chunk           |
| Forgetting to count skipped files    | LLM doesn't know it's seeing a partial view          |
| Pre-computing nothing                | LLM spends tokens on pattern matching it can't do well|

---

## ✅ Azure DevOps PR Example

**PR #42 has 47 changed files:**

```
Raw:       47 files, 8,200 lines  → ~65,000 tokens  ❌
Filtered:  22 files (removed 25 lock/dist/snap files)
Chunked:   22 files × avg 3 chunks = 66 chunks available
Summarized: 22 file summaries, each ~200 tokens = ~4,400 tokens ✅

Risk signals found:
- src/api/userController.ts  → "SQL injection risk", "TODO left in"
- src/config/settings.ts     → "Hardcoded secret"
- src/auth/middleware.ts      → "Auth bypass pattern"

LLM focuses review on these 3 files → precise, actionable feedback
```

---

## ✅ How I Will Use This in My MCP Project

- Build a `filter_changed_files()` utility used by all diff-related tools
- Set `MAX_LINES_PER_CHUNK = 400` as a server-wide constant
- Create two diff tools:
  - `get_pr_diff` → returns filtered file list + per-file summaries with risk signals
  - `get_pr_diff_chunk` → returns a specific chunk of a specific file on demand
- Pre-compute risk signals (SQL, secrets, eval, auth bypass) before returning data
- Always include `total_files`, `reviewed_files`, `skipped_files` in responses
- Truncate all text previews to 300 chars max

---

*Next: Section 5 — Real-World Implementation (Building the full MCP server in Node.js + Python)*