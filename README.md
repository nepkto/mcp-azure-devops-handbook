<div align="center">

# 🤖 MCP Azure DevOps Handbook

### A complete, hands-on guide to designing and building an MCP Server for Azure DevOps Pull Request Review

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green?logo=node.js)](https://nodejs.org)
[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python)](https://python.org)
[![MCP](https://img.shields.io/badge/MCP-Model%20Context%20Protocol-purple)](https://modelcontextprotocol.io)
[![Azure DevOps](https://img.shields.io/badge/Azure%20DevOps-API%20v7.1-blue?logo=azure-devops)](https://learn.microsoft.com/en-us/rest/api/azure/devops)
[![VS Code](https://img.shields.io/badge/VS%20Code-Compatible-007ACC?logo=visual-studio-code)](https://code.visualstudio.com)
[![GitHub Copilot](https://img.shields.io/badge/GitHub%20Copilot-Ready-black?logo=github)](https://github.com/features/copilot)
[![Stars](https://img.shields.io/github/stars/nepkto/mcp-azure-devops-handbook?style=social)](https://github.com/nepkto/mcp-azure-devops-handbook)

<br/>

> **"From zero to production-ready MCP Server — with real code, real patterns, and real Azure DevOps integration."**

[📖 Start Reading](#-learning-roadmap) · [🚀 Quick Start](#-quick-start) · [📁 Docs](#-documentation) · [🤝 Contribute](#-contributing)

</div>

---

## 🧠 What Is This?

This repository is a **structured, progressive learning handbook** for **Model Context Protocol (MCP)** — the open standard that lets LLMs like GitHub Copilot connect to external tools, APIs, and services.

The focus is **practical and real-world**: every concept is taught through the lens of building an **AI-powered Azure DevOps Pull Request Review Server**.

By studying this repo, you will:

- ✅ Understand MCP deeply — not just theoretically
- ✅ Design and build a production-ready MCP Server
- ✅ Integrate with Azure DevOps REST API (PR details, diffs, comments)
- ✅ Handle real engineering challenges (token limits, security, scaling)
- ✅ Have working code in **both Node.js (TypeScript) and Python**

---

## 🗺️ Learning Roadmap

| # | Section | Topics Covered | Status |
|---|---------|---------------|--------|
| 01 | [**Foundations**](docs/section-01-foundations.md) | What MCP is, why it exists, LLM + Client + Server | ✅ Done |
| 02 | [**Architecture Deep Dive**](docs/section-02-architecture.md) | Full request flow, transport, tool schema | ✅ Done |
| 03 | [**MCP Server Design**](docs/section-03-mcp-server-design.md) | Tool naming, input/output schema, good vs bad design | ✅ Done |
| 04 | [**Data Flow & Context**](docs/section-04-data-flow.md) | Token limits, chunking, filtering, summarization | ✅ Done |
| 05 | [**Real-World Implementation**](docs/section-05-implementation.md) | Full Node.js + Python MCP server, Azure DevOps client | ✅ Done |
| 06 | **Azure DevOps Use Case** | PR details, diffs, comments, structuring for LLM | 🔄 In Progress |
| 07 | **AI + MCP Interaction** | Prompt design, tool selection, high-quality PR reviews | 🔜 Coming |
| 08 | **Advanced Patterns** | Tool composition, `review_pr`, `detect_code_smells` | 🔜 Coming |
| 09 | **Performance & Scaling** | Caching, rate limiting, large PR handling | 🔜 Coming |
| 10 | **Security & Best Practices** | PAT security, input validation, data protection | 🔜 Coming |
| 11 | **Engineering Thinking** | Trade-offs, extensibility, maintainability | 🔜 Coming |
| 12 | **Final Summary** | Mental model, checklist, consolidated docs | 🔜 Coming |

---

## ⚡ Quick Start

### Prerequisites

- [Node.js 18+](https://nodejs.org) or [Python 3.11+](https://python.org)
- [VS Code](https://code.visualstudio.com) with [GitHub Copilot](https://github.com/features/copilot)
- An [Azure DevOps](https://dev.azure.com) account with a PAT token

### Node.js Setup

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/mcp-azure-devops-handbook.git
cd mcp-azure-devops-handbook

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your Azure DevOps credentials

# Build and run
npm run build
npm start
```

### Python Setup

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/mcp-azure-devops-handbook.git
cd mcp-azure-devops-handbook

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your Azure DevOps credentials

# Run
python -m src.server
```

### Environment Variables

```env
AZURE_DEVOPS_ORG=https://dev.azure.com/your-org
AZURE_DEVOPS_PAT=your_personal_access_token
AZURE_DEVOPS_PROJECT=YourProjectName
```

> ⚠️ **Never commit your `.env` file.** It's already in `.gitignore`.

---

## 🏗️ Architecture Overview

```
User Prompt (VS Code / GitHub Copilot)
         │
         ▼
   LLM (GitHub Copilot)
   • Reads tool list from MCP Server
   • Selects the right tool based on intent
   • Extracts arguments from user message
         │
         ▼ JSON-RPC (stdio)
   MCP Client (VS Code — built-in)
   • Routes tool calls to MCP Server
   • Manages server lifecycle
         │
         ▼ MCP Protocol
   MCP Server ← YOU BUILD THIS
   • Exposes: get_pr_details, get_pr_diff,
              post_review_comment, review_pr...
   • Validates inputs
   • Filters + summarizes data
         │
         ▼ HTTPS / REST (PAT Auth)
   Azure DevOps API
   • PR metadata, diffs, comments, files
```

---

## 📁 Documentation

All documentation is structured for **reuse as internal wiki or project README**:

| File | Description |
|------|-------------|
| [`docs/section-01-foundations.md`](docs/section-01-foundations.md) | MCP fundamentals, the USB-C analogy, 3-player model |
| [`docs/section-02-architecture.md`](docs/section-02-architecture.md) | Step-by-step request flow, JSON-RPC, transport options |
| [`docs/section-03-mcp-server-design.md`](docs/section-03-mcp-server-design.md) | Tool naming, schema design, good vs bad examples |
| [`docs/section-04-data-flow.md`](docs/section-04-data-flow.md) | Token budgets, filter/chunk/summarize pipeline |
| [`docs/section-05-implementation.md`](docs/section-05-implementation.md) | Full server setup, Azure DevOps client, VS Code wiring |

Each doc includes:
- ✅ Summary
- ✅ Key Concepts
- ✅ Code Snippets (Node.js + Python)
- ✅ Design Guidelines
- ✅ Common Mistakes
- ✅ Azure DevOps PR Example

---

## 🛠️ Tools Implemented (So Far)

| Tool | Description | Status |
|------|-------------|--------|
| `get_pr_details` | Fetch PR metadata: title, author, status, reviewers | ✅ |
| `get_pr_diff` | Fetch changed files with filtering and risk signals | ✅ |
| `get_pr_diff_chunk` | Fetch specific diff chunk for large files | ✅ |
| `get_pr_comments` | Fetch existing review comments | 🔄 |
| `post_review_comment` | Post AI-generated review feedback | 🔄 |
| `review_pr` | Orchestrate full PR review (advanced) | 🔜 |
| `detect_code_smells` | Detect security/quality issues in diff | 🔜 |

---

## 💡 Key Engineering Patterns

```
🔒 Security          → PAT via .env, never in code, validated at startup
🧹 Data Pipeline     → Filter → Chunk → Summarize before LLM sees data
📝 Tool Design       → verb_noun names, rich descriptions, typed schemas
⚠️  Error Handling   → Structured JSON errors, never unhandled exceptions
📤 stdout Rule       → stdout = MCP protocol only. Logs → stderr always
🔌 Modular Design    → One tool per file, shared Azure client singleton
```

---

## 🤝 Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Ideas for contribution:
- Add new Azure DevOps tools
- Add tests
- Improve error messages
- Add support for GitHub / GitLab alongside Azure DevOps
- Add Docker support

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---

## 👤 Author

**Anil Baniya**
- GitHub: [@nepkto](https://github.com/nepkto)
- LinkedIn: [Anil Baniya](https://linkedin.com/in/anil-baniya)

---

<div align="center">

⭐ **If this helped you, please star the repo — it helps others find it!** ⭐

</div>
