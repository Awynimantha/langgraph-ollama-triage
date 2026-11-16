# langgraph-ollama-triage

A **local AI agent** that triages Java / WSO2 product logs and stack traces.
Built with [LangGraph](https://langchain-ai.github.io/langgraph/) for the agent loop and [Ollama](https://ollama.com) for a model that runs entirely on your machine — no data leaves your laptop.

Give it a log file or a stack trace, and the agent will:

1. **Find** the relevant error and its stack trace in the log
2. **Locate** the classes in the trace inside a local source checkout
3. **Read** the relevant source and docs
4. **Explain** the likely root cause — as a hypothesis with the evidence it used — and suggest next steps

> **Status:** 🚧 Early development / learning project. See the [Roadmap](#roadmap).

---

## Why

- **Private by default** — runs on a local model through Ollama, so logs and configs never go to a cloud LLM.
- **Explicit control flow** — LangGraph models the agent as a graph (model → tools → model …), so every step can be inspected, tested and interrupted.
- **Learning by building** — a small, readable codebase covering the core agent concepts: the ReAct loop, tool design, state, memory and human-in-the-loop.

---

## How it works

```
                ┌──────────────────────────────┐
  user input ─► │  agent (LLM via Ollama)      │ ◄──────────────┐
 (log / trace)  │  decides: answer or call tool│                │
                └──────────────┬───────────────┘                │
                               │ tool call                      │ tool result
                               ▼                                │
                ┌──────────────────────────────┐                │
                │  tools                       │ ───────────────┘
                │  • read_log / grep_log       │
                │  • find_class / read_source  │
                │  • search_docs               │
                └──────────────────────────────┘
                               │ no more tool calls
                               ▼
                     triage report (cause, evidence,
                     confidence, next steps)
```

| Component | Location | Purpose |
|---|---|---|
| Graph | `src/triage_agent/graph.py` | Builds the LangGraph state machine (agent node ↔ tool node loop) |
| State | `src/triage_agent/state.py` | Conversation messages and working data passed between nodes |
| Prompts | `src/triage_agent/prompts.py` | System prompt that defines the triage behaviour and report format |
| Tools | `src/triage_agent/tools/` | Functions the model can call: logs, source code, docs |
| Config | `src/triage_agent/config.py` | Model name, Ollama URL, workspace paths (from `.env`) |
| CLI | `src/triage_agent/main.py` | Entry point: `triage --log <file>` / `triage --trace <file>` |

---

## Tech stack

| | |
|---|---|
| Agent framework | [LangGraph](https://langchain-ai.github.io/langgraph/) |
| LLM runtime | [Ollama](https://ollama.com) (local) |
| LLM integration | [`langchain-ollama`](https://python.langchain.com/docs/integrations/chat/ollama/) (`ChatOllama` + `.bind_tools()`) |
| Language | Python 3.11+ |
| Tests | pytest |

---

## Prerequisites

1. **Python 3.11+**
2. **Ollama** installed and running — see the [Ollama repo](https://github.com/ollama/ollama)
3. A model that **supports tool calling** — browse [ollama.com/search?c=tools](https://ollama.com/search?c=tools) and pick one that fits your RAM/GPU, e.g.:
   ```bash
   ollama pull qwen2.5:7b
   ```
4. *(Optional)* A local checkout of the source code you want the agent to trace into, placed under `workspace/repos/`.

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/Awynimantha/langgraph-ollama-triage.git
cd langgraph-ollama-triage

# 2. Create a virtual environment and install
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e ".[dev]"

# 3. Configure
cp .env.example .env               # then edit model name / paths if needed

# 4. Make sure Ollama is running and the model is pulled
ollama serve &                     # if not already running
ollama pull qwen2.5:7b

# 5. Run against a sample
triage --log samples/logs/sample-error.log
```

---

## Configuration

All settings come from environment variables (or a `.env` file). See [`.env.example`](.env.example).

| Variable | Default | Description |
|---|---|---|
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama server URL |
| `OLLAMA_MODEL` | `qwen2.5:7b` | Model name (must support tool calling) |
| `OLLAMA_TEMPERATURE` | `0` | Keep low for consistent analysis |
| `WORKSPACE_DIR` | `./workspace` | Root for local logs, repos and docs |
| `SOURCE_DIR` | `./workspace/repos` | Source checkouts the agent can search |
| `DOCS_DIR` | `./workspace/docs` | Markdown docs the agent can search |
| `MAX_TOOL_CALLS` | `15` | Safety limit on agent loop iterations |

---

## Project structure

Planned layout (files are added as each roadmap step is implemented):

```
langgraph-ollama-triage/
├── README.md
├── pyproject.toml             # dependencies and the `triage` CLI entry point
├── .env.example               # sample configuration
├── .gitignore
├── docs/
│   └── architecture.md        # design notes and decisions
├── samples/                   # SANITIZED example inputs only
│   ├── logs/
│   │   └── sample-error.log
│   └── stacktraces/
├── src/
│   └── triage_agent/
│       ├── __init__.py
│       ├── main.py            # CLI
│       ├── config.py          # settings
│       ├── state.py           # graph state
│       ├── prompts.py         # system prompt
│       ├── graph.py           # LangGraph wiring
│       └── tools/
│           ├── __init__.py    # exports ALL_TOOLS
│           ├── logs.py        # read_log, grep_log, extract_stacktraces
│           ├── source.py      # find_class, read_source
│           └── docs.py        # search_docs
├── tests/
│   └── test_tools.py
└── workspace/                 # local-only, git-ignored (logs, repos, docs)
```

---

## Tools

| Tool | Signature | What it does |
|---|---|---|
| `read_log` | `read_log(path, start_line=0, max_lines=200)` | Reads a window of a log file |
| `grep_log` | `grep_log(path, pattern, context=5)` | Finds lines matching a regex, with surrounding context |
| `extract_stacktraces` | `extract_stacktraces(path)` | Pulls out `Exception` / `Caused by` blocks |
| `find_class` | `find_class(fqcn)` | Locates a Java class file under `SOURCE_DIR` |
| `read_source` | `read_source(path, line, context=40)` | Reads source around a line number from a stack frame |
| `search_docs` | `search_docs(query)` | Keyword search over markdown files under `DOCS_DIR` |

Tool design guidelines used in this project:
- Small, single-purpose tools with **clear names and descriptions** — the model picks tools based on these.
- **Bounded output** (line windows, max matches) to keep the model's context small.
- **Read-only.** Any future "write" action must go through a human-approval step.

---

## Roadmap

- [ ] **v0.1** — ReAct loop with `create_react_agent`, log tools, CLI
- [ ] **v0.2** — Source tools: map stack frames → local source files
- [ ] **v0.3** — Custom `StateGraph` (explicit nodes: extract → locate → analyse → report)
- [ ] **v0.4** — Docs search tool + structured triage report (cause / evidence / confidence / next steps)
- [ ] **v0.5** — Human-in-the-loop approval and checkpointing (pause / resume)
- [ ] **v0.6** — Load tools from MCP servers via `langchain-mcp-adapters`
- [ ] **v0.7** — Evaluation set: sample logs + expected findings, to compare models and prompts

---

## Data privacy

This project is designed to run **locally**, but you are still responsible for what you put in the repository:

- ❌ **Never commit real customer logs, configs, hostnames, IPs, credentials or tokens.**
- ✅ Keep real inputs under `workspace/` — it is git-ignored.
- ✅ Only commit **sanitized** examples under `samples/` (replace emails, IPs, hostnames, secrets with placeholders such as `[EMAIL]`, `[IP]`, `[REDACTED]`).

---

## Learning references

- Anthropic — [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
- Anthropic — [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Lilian Weng — [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents)
- [Hugging Face Agents Course](https://huggingface.co/learn/agents-course)
- [LangChain Academy — Introduction to LangGraph](https://academy.langchain.com)
- [Model Context Protocol](https://modelcontextprotocol.io)

---

## License

TBD.
