# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Package manager:** `uv`

```bash
# Install dependencies (development)
uv sync --dev

# Run all tests
uv run pytest -v --timeout=30

# Run a single test file
uv run pytest tests/test_foo.py -v

# Run a single test by name
uv run pytest tests/test_foo.py::test_bar -v

# Lint
uv run ruff check .

# Build distribution
uv build

# Run the CLI
uv run EvoSci
```

## Architecture

EvoScientist is a multi-agent AI framework for automated end-to-end scientific research. It uses a **human-on-the-loop** paradigm: the user provides a research question; agents plan, code, debug, analyze, and write a paper autonomously.

### Agent Graph (`EvoScientist/EvoScientist.py`)

The core orchestrator builds a LangGraph-based agent graph with lazy initialization (defers heavy imports). It coordinates six specialized sub-agents defined in `EvoScientist/subagent.yaml`:

- **Planner** — Creates experimental plans (no web search, no code execution)
- **Research** — Conducts web research via Tavily API
- **Code** — Implements and runs reproducible experiment scripts
- **Debug** — Reproduces failures and applies minimal fixes
- **Data-Analysis** — Computes metrics and generates plots
- **Writing** — Drafts paper-ready Markdown reports

### Key Subsystems

| Directory | Purpose |
|-----------|---------|
| `EvoScientist/llm/` | Multi-provider LLM abstraction (Anthropic, OpenAI, Google, NVIDIA, Ollama) |
| `EvoScientist/cli/` | CLI (`typer`) and TUI (`textual`) interfaces |
| `EvoScientist/channels/` | Messaging integrations (Telegram, Slack, Discord, WeChat, etc.) |
| `EvoScientist/middleware/` | Persistent memory and error handling layers |
| `EvoScientist/mcp/` | Model Context Protocol integration for external tools |
| `EvoScientist/skills/` | 200+ predefined research skills + custom skill support |
| `EvoScientist/stream/` | Event streaming and Rich terminal display |
| `EvoScientist/tools/` | Core tools: search, think, execute |
| `EvoScientist/backends.py` | Sandbox and file system backends for workspace I/O |
| `EvoScientist/prompts.py` | System prompts for the main agent |
| `EvoScientist/config/` | Configuration management (`~/.config/evoscientist/config.yaml`) |

### Data Flow

```
User Input (CLI / TUI / Channel)
  → Main Agent Graph (EvoScientist.py)
  → LLM call with system prompt + history
  → Sub-agent delegation (via subagent.yaml roles)
  → Tool execution (search, code, analyze, write)
  → Event stream → Rich/Textual display update
  → State checkpointed to SQLite
```

### State Persistence

Sessions are checkpointed to SQLite. The workspace (experiment files, plots, reports) is managed through the sandbox backend in `backends.py`. User config lives in `~/.config/evoscientist/config.yaml` and is created by `EvoSci onboard`.

### LLM Provider Selection

Provider is chosen at startup via config or CLI flags. The `EvoScientist/llm/` module wraps `langchain-anthropic`, `langchain-openai`, `langchain-google-genai`, `langchain-nvidia-ai-endpoints`, and `langchain-ollama`. Required API keys are documented in `.env.example`.

### Testing

Tests live in `tests/` (36 files, ~890 tests). They are isolated and do not require real API credentials. CI runs on Python 3.11 and 3.12 with a 30-second per-test timeout.
