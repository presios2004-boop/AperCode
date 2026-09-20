# AperCode

[日本語](README.md)

A general-purpose coding agent that runs on local LLMs (Ollama), with a browser/desktop UI.
Built for medium-sized projects: context compaction, sub-agents, and an experience memory that learns from past successes and failures.

## Features

- **Agent loop with tools**: read/write/edit files, search, run shell commands, with per-tool approval (allow / ask / deny)
- **Local first**: works with any Ollama model that supports tool calling (a JSON fallback is available for models that don't); Anthropic API is optional
- **Context compaction**: when the history approaches the context window, older messages are summarized automatically
- **Sub-agents**: delegate exploration (read-only) or implementation (read/write) tasks to child agents with their own context
- **Experience memory**: lessons are extracted after each task and stored in SQLite (FTS5 + embedding vectors, hybrid search). Relevant lessons are injected into the prompt on every request. Domain-agnostic — tags are inferred from the project.
- **Attachments**: paste images / long text from the clipboard, drag & drop files, right-click paste
- **Desktop app**: `app.py` starts the server and opens a dedicated window (pywebview → Chromium `--app` → default browser)
- **MetaTrader 5 (wine)**: compile MQL5 and run the strategy tester as tools (optional)

## Quick start

```bash
git clone <this repo> apercode && cd apercode
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp config.example.yaml config.yaml      # edit model name, workspace root, etc.
ollama pull qwen3:8b bge-m3             # chat model + embedding model
python3 app.py                          # or: python3 main.py and open http://127.0.0.1:8765
```

Linux desktop launcher (app menu + desktop icon):

```bash
bash install_desktop.sh
pip install "pywebview[qt]"             # optional: dedicated window (Debian/Ubuntu: sudo apt install libxcb-cursor0)
```

## Configuration

Settings are merged in this order (later wins):
1. `config.example.yaml` (bundled defaults)
2. `config.yaml` (project root, git-ignored)
3. `~/.apercode/config.yaml` (per-user overrides)

Key settings: `llm.model`, `llm.num_ctx`, `workspace.root` (the only directory tree the agent may touch),
`permissions.*`, `compaction.*`, `subagents.*`, `memory.*`, `mt5.*`.

User data (sessions, memory DB, logs) lives in `~/.apercode/`.

## Layout

```
app.py                  desktop launcher
main.py                 server only
codeagent/agent.py      agent loop, compaction, sub-agents, memory recall/reflection
codeagent/llm.py        Ollama / Anthropic client
codeagent/memory/       experience memory store
codeagent/tools/        fs, shell, mt5, subagent, memory tools
codeagent/server/       FastAPI + WebSocket, static UI
```

## License

MIT
