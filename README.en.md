# AperCode

[日本語](README.md)

A general-purpose coding agent that runs on local LLMs (Ollama), with a browser/desktop UI.
Built for medium-sized projects: context compaction, sub-agents, and an experience memory that learns from past successes and failures.

## Language

The UI, dialogs, verification messages and tool descriptions are available in **English and Japanese**. The default is `auto`
(follows the OS `LANG`); change it with the language selector at the bottom of the sidebar or `ui.language: en` in
`~/.apercode/config.yaml`. The system prompt has an English version too (`llm.prompt_language`, `auto` follows the UI
language); the model replies in the language you write in.

## Features

- **Agent loop with tools**: read/write/edit files, search, run shell commands, with per-tool approval (allow / ask / deny). "Always allow for this session" covers code generation, edits and test runs; destructive commands (`rm`, `git reset --hard`, `sudo`, `dd`, `kill`, `drop table`, …) always ask
- **Local first**: works with any Ollama model that supports tool calling (a JSON fallback is available for models that don't); Anthropic API is optional
- **Context compaction**: when the history approaches the context window, older messages are summarized automatically
- **Sub-agents**: delegate exploration (read-only) or implementation (read/write) tasks to child agents with their own context. Since v1.2.2 the UI has two model selectors: the main model (chat/implementation) and a sub model used for sub-agents and code review — pick a small model (e.g. `llama3.1:8b`) there to make exploration and review faster; the choice is remembered in `~/.apercode/state.json`
- **Experience memory**: lessons are extracted after each task and stored in SQLite (FTS5 + embedding vectors, hybrid search). Relevant lessons are injected into the prompt on every request. Domain-agnostic — tags are inferred from the project.
- **Automatic context sizing**: at startup and on model switch, the free VRAM (or RAM) and the model's layer/KV-head geometry are used to pick a `num_ctx` that actually fits (capped by `llm.num_ctx`; tunable via `llm.auto_ctx.*`, incl. `kv_cache_type` for q8_0 KV cache) With multiple GPUs, free VRAM is summed across all of them.
- **Verification layer (v1.2.0)**: every `write_file`/`edit_file` is followed by a syntax/lint/type check for that language (`py_compile`+`ruff`, `node --check`/`tsc`, `go vet`, `cargo check`, `shellcheck`, …; external tools are used only if installed) and failures are handed back to the model to fix. Before the final answer the project's tests are run automatically (`pytest`/`unittest`/`npm test`/`go test`/`cargo test`/`make test`, auto-detected; a `run_tests` tool is also available) and failures are pushed back, up to `verify.max_fix_rounds` times; projects without tests are asked to add minimal ones. Changes above `verify.review.min_changed_lines` are reviewed by a read-only sub-agent that reads the diff and reports high/medium issues for the parent to fix. A verification summary is shown at the end of each turn. **Tests are protected against assertion weakening (v1.2.1)**: once a test run has failed in a turn, edits to existing test files (and shell commands that rewrite them) are excluded from "always allow" and require explicit approval (`verify.lock_tests: ask`) or are rejected (`deny`); adding new tests stays allowed, and any change to pre-existing tests is flagged in the summary.
- **Cloud consultation (v1.2.4)**: when the local model is still stuck after web search, an `ask_cloud` tool sends only the goal, the error, what was tried and short code excerpts to a cloud model (Anthropic, Gemini, OpenAI, DeepSeek, Groq, Mistral, OpenRouter, xAI or any OpenAI-compatible URL) and gets back hypotheses and a fix proposal; the local agent still implements and verifies. Gated behind a minimum number of failures, capped per turn, approval before sending, secrets masked. Configured from the "クラウド相談" dialog (stored in `~/.apercode/config.yaml`) or `cloud.*`
- **Browser (v1.4.0)**: with Playwright installed (`pip install playwright && python -m playwright install chromium`; optional) the agent gets `browser_open` / `browser_read` / `browser_click` / `browser_type` / `browser_press` / `browser_scroll` / `browser_eval` / `browser_screenshot` / `browser_console` / `browser_close` and is told to verify web apps it built in a real browser (console errors, DOM existence, screenshot) before reporting done. Screenshots are shown in the chat and passed to vision-capable models as images. `browser.mode: headed` (default, a visible window) or `headless`; `browser.channel: chromium | chrome | msedge | firefox` to use an installed browser; `browser.cdp_url` to attach to a running Chrome (`--remote-debugging-port=9222`); sites other than `localhost` ask for approval (`browser.external_sites`). A "ブラウザ / Browser" button in the sidebar (v1.4.1) launches the same browser at a URL you enter, and the LLM continues from that page
- **Helper tools (v1.5.0)**: `repl` (a persistent Python / Node session — variables and imports survive across calls; uses the project's `.venv`), `process_start` / `process_status` / `process_logs` / `process_stop` (background servers with logged output, optional wait-for-port, stopped on exit), `read_document` (PDF / Word / Excel / PowerPoint / OpenDocument / CSV to text, no extra libraries needed except for PDF) and `check_js` (JavaScript / HTML checker: syntax, unresolvable `import` / `require` and `<script src>`, plus a real-browser run that collects JS exceptions, `console.error` and failed requests; `probes` evaluate JS expressions such as `ForceGraph3D.toString()` to verify how a library is initialised, and `libraries` loads CDN URLs into a blank page to inspect an API before writing code). `.html` files are now part of the automatic post-edit verification
- **Stuck detection**: repeated identical tool calls or consecutive failures trigger a nudge to research (memory → web search) and change approach; hitting the tool-call limit produces a status report instead of silence, and "continue" resumes
- **Web search when stuck**: `web_search` (DuckDuckGo → Bing → Brave → Mojeek fallback chain, no API key) and `fetch_url` tools; the system prompt tells the agent to search, read docs/issues, try fixes, and save what worked to memory with the source URL instead of giving up
- **Automatic backups**: before `write_file`/`edit_file` touches an existing file, the original is copied to `backup/<timestamp>/` in the project (one folder per user request). Restore with the `restore_backup` tool or a plain `cp -r`
- **Icon generator**: `make_icon` tool / `python3 -m codeagent.tools.icon` produces PNG (16–512), `.ico` and `.svg` app icons (Pillow)
- **Embedding recovery and Ollama health (v1.6.2)**: lessons saved while the embedding model (bge-m3) is unavailable are stored without vectors and vectorised automatically once it is back; a banner warns when Ollama lacks the chat or embedding model, or has no models at all (another Ollama, e.g. a host `ollama serve`, occupying port 11434)
- **Background summarisation by the sub LLM (v1.6.1)**: when the history reaches 75 % of the compaction threshold, the sub LLM starts summarising older history in the background while the main LLM keeps working; when compaction is due the prepared summary is swapped in with no wait. The sub LLM runs on GPU only if it fits in free VRAM, otherwise on CPU (`compaction.bg_device: auto`), so the main model is never evicted. Lesson extraction also runs on the sub LLM
- **Prompt-injection defense (v2.2.0)**: reading external content (web search, fetched pages, browser, peers, cloud AI) marks the session as tainted, shown as a red "⚠ External content" chip. While tainted, file changes, commands, sending (`ask_peer`, `cloud_chat`), GUI input and memory saves always ask for approval even when "always allow" was granted; a red warning blinks in the chat, and keeps blinking when the content contains instruction-like text ("ignore previous instructions", "you are now", "send the API key"...). Lessons saved while tainted are tagged `untrusted` and are not injected into prompts until verified in the memory dialog. `fetch_url` / `browser_open` to URLs with secret-looking strings, long queries or domains not seen in search results or user messages ask first. External text is stripped of invisible characters and hidden HTML (`display:none`, `hidden`, `aria-hidden`) and wrapped in `<external_content>` as data, with a system-prompt rule not to follow instructions inside. Click the chip to review the sources and clear it (`security.taint_guard`, `security.wrap_external`). Search results alone (short snippets) and sites in `security.trusted_domains` (official docs such as Python, MDN, Node.js, Rust, Go by default) do not taint the session unless instruction-like text is found (v2.2.1)
- **Self-protection and command sandbox (v2.5.0)** — ⚠ do not let AperCode modify AperCode itself: a mistake by the LLM or instructions injected through web pages or other AIs could break its own safety checks, so changes to AperCode are made by the user (or maintainer) outside AperCode. `write_file` / `edit_file` / `restore_backup` etc. targeting the install directory or `~/.apercode` (config, API keys, sessions, memory) are refused, even if the working directory is inside it. Give AperCode a dedicated working directory with its own virtual environment: set it in Settings → "AperCode workspace" → "Working directory" (default `~/Documents/Aper_work`, `workspace.default_project`; it cannot be inside AperCode itself). If the folder or its `.venv` is missing, a banner at the top offers "Create", which makes both (manually: `python3 -m venv ~/Documents/Aper_work/.venv`; on Ubuntu `sudo apt install python3-venv` may be needed) (v2.6.0); a `.venv` / `venv` in the working directory is put first on PATH for commands, tests and `repl`. On Linux with bubblewrap (`sudo apt install bubblewrap`), `run_command`, `process_start`, `repl`, tests and auto-checks run in a sandbox where only the working directory, `/tmp` and `sandbox.writable` (default `~/.cache`, `~/.npm`) are writable, AperCode itself is read-only and `~/.apercode` is hidden; the status is shown in the header ("🛡 Sandboxed" / "⚠ Not sandboxed", click for Settings) and in Settings, and when bubblewrap is missing a banner asks to `sudo apt install bubblewrap` (re-checked automatically after installing) (v2.6.0). `sandbox.mode`: `auto` (default, use if available) / `on` (required; commands are refused without it) / `off` (also in the settings dialog); `sandbox.network: false` cuts network access. Writes outside the working directory such as `pip install --user` fail — install into the project `.venv`. Without the sandbox, commands that mention AperCode's own paths always ask. Desktop-app tools (`gui_launch` …) are not sandboxed
- **Chat and work modes (v2.1.0)**: sessions start as a conversation (`Auto`, `modes.default`): questions and research are answered directly, without a task brief or plan. When the LLM tries to change files or run commands, a "🛠 Start working?" card asks for a working directory; choosing it builds the task brief and plan from the conversation so far and enters the work phases (the action itself is not run yet), while "Keep chatting" makes it answer in the conversation. After the work is done it returns to chat. The header selector switches Auto / 💬 Chat (never changes files) / 🛠 Work (always work, the v2.0 behaviour). Approval cards gain **Pause (ask a question)**: the action is neither run nor denied, the turn stops, your next message is answered as a question and the LLM asks before retrying
- **Task brief (v1.8.0)**: the first instruction is rewritten into a Markdown brief (goal, requirements, constraints, done-when) at `<project>/.apercode/briefs/<session>.md`; AperCode re-reads it and adds it to the system prompt on every LLM request, so the goal survives long work and history compaction. Follow-up instructions are appended; the LLM deletes it with `finish_brief` when the goal is achieved (refused while tests or auto-checks fail). View / edit / delete it from the "📋 Brief" chip in the header
- **Work phases and checkpoints (v1.9.0)**: AperCode, not the LLM, drives plan → implement → verify → fix → investigate → done check, switching a short instruction and the available tools per phase. Plan: read-only tools, then `set_plan` registers 3-7 steps (a plain question is just answered). Implement: only the current step, then `step_done`. Verify: AperCode runs the auto-checks and tests itself. Fix: only the reported error; after `phases.max_fix_rounds` (3) the files are rolled back to the last checkpoint and the failed attempts are removed from the LLM history, then Investigate (read-only, web search, cloud consult; no file changes) writes a different approach and the step is retried. After `phases.max_investigations` (1) it reports and stops; "continue" resumes the same step. A checkpoint of the files AperCode changed is saved at the start and after every verified step in `<project>/.apercode/checkpoints/<session>/`; the "🧭" chip in the header shows the phase and step and lets you roll back to any checkpoint (current files go to backup/ first). Changes made through `run_command` are not tracked. If planning needs information the LLM cannot get (e.g. an unreadable folder), it asks you and stops; your reply, however short, continues the plan phase (v1.9.1). Steps that tests cannot check (commands, GUI operations) are not done just because the action ran: AperCode asks for the result to be observed and accepts the step only after an observation and a `step_done` evidence saying what was seen (v1.10.1, `phases.require_evidence`); if the result was already observed after the last action and evidence is given, the step passes at once without another screenshot (v1.11.1). The done check also looks at the last screen or output observed (e.g. the other party's reply) for requests or questions still unanswered (v1.11.2)
- **Stronger verification (v1.10.0)**: a built-in structure check (no extra tools) flags `self.xxx` references that are defined nowhere in the class and calls whose argument count or keyword names do not match the definition, reporting only problems the edit introduced; if `pyright` or `mypy` is installed, run-time-crashing errors (missing attributes, undefined names, bad calls) on the changed lines are reported too; if `coverage` is installed, tests run under `coverage run -m pytest` and changed lines that no test executed trigger a one-time request to add tests (`verify.structure`, `verify.types`, `verify.coverage`)
- **Harness regression tests and benchmark (v1.10.0)**: `python -m pytest` runs the harness test suite with a scripted fake LLM (phases, tool gating, verification, rollback, checkpoints, structure check, coverage) in seconds. `python bench/run_bench.py` has the real local model solve 10 small tasks judged by hidden acceptance checks (the test-writing task is judged by mutation testing), appends success rate, time, tool calls, fix rounds, rollbacks and tokens to `bench/results.jsonl` and compares with the previous run of the same model (`--repeat`, `--phases off`, `--history`; `bench/validate_tasks.py` checks new tasks)
- **Desktop app tools (v1.11.0, Linux / X11)**: `gui_windows`, `gui_launch` (start an app and wait for its window; no second copy), `gui_activate`, `screenshot` (the window or screen is passed to the model as an image directly) and `gui_input` (bring to front, then click / paste text incl. Japanese via the clipboard / press keys). Click coordinates are pixels on the last screenshot and are mapped with the current window position, so moved windows still work; screenshots crop a full-screen capture (Electron windows can come out blank when captured alone); input uses xdotool, xte or python-xlib, whichever works. After sending text with Enter, `gui_input` waits `gui.reply_wait` (10 s) before returning, and `digest` records the other party's state (replied / replying / waiting for its user's approval): nothing is sent while it is still writing or waiting for approval, and its approval buttons are never clicked (v2.4.0). Needs `wmctrl x11-utils xclip imagemagick` plus one of `xdotool` / `xautomation` / `python3-xlib`. After an approval dialog, focus is returned to the window that was active before (`permissions.restore_focus`). Runaway thinking is cut after `llm.think_limit_chars` (20,000) and the next call runs without thinking. Phases: `set_plan` takes `kind` (`code` / `operate`); hints during work no longer force a re-plan (the model may call `set_plan` mid-work instead); "start over" makes a fresh plan; steps that changed no files retry another way instead of rollback + read-only investigation; observing tools are not counted as repeats; newly written scripts must be run once before a step passes
- **Working with other AIs (v2.0.0)**: `set_plan` gets `kind: converse` for conversations with another AI or a person. After a reply arrives, AperCode requires `digest` before the next message is sent (chat-app `gui_input` text, `ask_peer` / `peer_reply` / `cloud_chat`); `digest` records the other party's points, advice and corrections, every question (including earlier unanswered ones), the reply plan and the insights to keep (saved to experience memory). When the user gives a number of messages, it is recorded as a guide (`set_plan` `rounds`); sending beyond it shows "💬 Continue the conversation?" with the reason taken from the digest (unanswered questions, reply plan) — allow sends that one message, deny stops and the final report notes the open question (v2.3.0). Unanswered questions stay in the implement-phase prompt, and the done check asks once to digest an unprocessed last reply. **Brief gap check** (`brief.clarify`): right after the task brief is written, AperCode checks the done/stop condition, target, scope of irreversible operations and deliverable; critical gaps are asked before any work starts (the reply, however short, is appended to the brief and the work starts), minor ones and methods (how to launch an app, which tools) are written into the brief as assumptions, and a stop condition that depends only on the other party gets a fallback limit as an assumption (v2.0.1) (`assume` never asks, `off` skips the check). `digest` runs once per reply, and waiting with `sleep` is not counted as a repeated call (v2.0.1). **Peers**: another AperCode on the LAN can take work through a token-protected peer API on its own port (`peer.*`; only the peer endpoints are exposed, nothing starts without `peer.token`); the requesting side lists them in `peers:` and gets `peer_list`, `ask_peer` (task, project, `mode: code|explore`, `files` delivered to the peer's `.apercode/inbox/`), `peer_reply`, `peer_status`, `peer_read_file` and `peer_verify` (runs the auto-detected tests on the peer). Only projects listed in `peer.projects` are accepted (the API does not start without it); AperCode's data (`~/.apercode`: config, API keys, sessions) and dot folders / files (`.ssh`, `.env`, …) are always refused. The peer works with its own LLM, tools and permissions; approvals follow `peer.on_approval` (`deny` / `allow_edit` / `allow`), and command execution, GUI, browser and `fetch_url` always go through that policy even when the local permission is allow; peer tools and `cloud_chat` are unavailable inside a peer request (sub-agents included); arbitrary commands are never accepted; a port that cannot be bound only prints a warning; delegated sessions appear in the peer's session list. `cloud_chat` talks to the cloud AI without the stuck gate of `ask_cloud`. Plain HTTP: use on a trusted LAN only
- **In-app browser (v1.7.0)**: with `browser.mode: panel` (default) the Playwright browser is shown in a side panel inside the AperCode window instead of a separate window (CDP screencast). The LLM's browser actions are visible live, and you can click, scroll, type (incl. IME and paste), enter URLs and go back/forward/reload in the panel; the panel opens automatically when the LLM starts using the browser. `headed` restores the separate window
- **Settings dialog (v1.6.0)**: change LLM parameters (thinking mode `think`, `num_predict`, temperature, timeout, keep_alive, tool mode, tool-call limit), context sizing and compaction, stuck-detection thresholds, cloud-consult conditions, verification / review / sub-agent / browser options from the UI; saved to `~/.apercode/config.yaml` and applied immediately, with "reset all" back to `config.yaml`. Model thinking (Ollama `thinking`) is shown collapsed in the chat; a reply that ends after thinking only is bounced back with "conclude or call a tool". For coder models that write code or tool-call JSON in the chat instead of calling tools: JSON tool calls in the text are parsed and executed even in native mode, and code-only replies are bounced back once with "write it with write_file / edit_file"
- **Colour-coded chat** (v1.5.1): main-LLM replies (blue), sub-LLM agents / reviews (purple, with a `sub LLM · model · mode` badge), web searches (green) and cloud consultations (orange, with provider / model) are distinguishable at a glance; a legend sits above the log
- **Code blocks with copy button** (v1.2.3): fenced code in replies is rendered with a language label and a Copy button; right-click in the chat gives "copy selection / copy code block / copy message"
- **Attachments**: paste images / long text from the clipboard, drag & drop files, right-click paste
- **Desktop app**: `app.py` starts the server and opens a dedicated window (pywebview → Chromium `--app` → default browser)
- **MetaTrader 5 (wine)**: compile MQL5 and run the strategy tester as tools (optional)

## Quick start

### Which installer to run first (Linux, from the zip)

There are two scripts with different jobs. Run them in this order, from the AperCode folder:

| Order | Script | What it does | Needed? |
|---|---|---|---|
| 1 | `setup_linux_window.sh` | installs GTK / WebKit2GTK via apt (dedicated window with native IME), creates AperCode's own `.venv` and installs `requirements.txt` | recommended (asks for your sudo password) |
| 2 | `install_desktop.sh` | only registers "AperCode" in the app menu (Development) and on the desktop; needs `.venv`, so run it after 1 | optional (to launch from the menu) |

```bash
cd AperCode                       # the folder extracted from the zip
bash setup_linux_window.sh        # 1. dependencies and .venv
bash install_desktop.sh           # 2. app menu / desktop launcher
sudo apt install bubblewrap       # recommended: command sandbox (v2.5.0)
ollama pull qwen3:8b bge-m3       # chat model + embedding model
```

- Without 1 (no apt, no IME needed), create `.venv` by hand instead: `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`, then run 2
- `setup_linux_window.sh` recreates `.venv` every time; running it later does not require redoing 2
- The scripts are Linux-only. On Windows / macOS follow the manual steps below and start with `python3 app.py`
- On first start, if a banner says the working directory is missing, press "Create" (it makes `~/Documents/Aper_work`, the folder AperCode works in, with its own `.venv`)
- There are two `.venv`s: the one in the AperCode folder **runs AperCode itself**, the one in the working directory **runs the code AperCode writes**. AperCode's `pip install` goes into the working-directory one

### Manual setup

```bash
git clone <this repo> apercode && cd apercode
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m playwright install chromium   # optional: browser tools (v1.4.0)
cp config.example.yaml config.yaml      # edit model name, workspace root, etc.
ollama pull qwen3:8b bge-m3             # chat model + embedding model
python3 app.py                          # or: python3 main.py and open http://127.0.0.1:8765
```

Linux desktop launcher (app menu + desktop icon):

```bash
bash install_desktop.sh
bash setup_linux_window.sh              # optional: dedicated GTK window with native IME support (recommended on Linux)
# or: pip install "pywebview[qt]"        # Qt window (no fcitx IME support in pip Qt)
```

## Troubleshooting

**Selecting a model gives "LLM server error 400 (the model name may be wrong: …)"**

If the name is correct, the model almost certainly **does not support tool calling in Ollama** (e.g. `deepseek-coder-v2:16b`,
`codellama`, older `gemma2`). Ollama answers `400 model does not support tools` when `tools` are sent to such a model.
Pick a model with the **tools** tag on its Ollama page (for coding: `qwen2.5-coder:14b` / `qwen2.5-coder:32b`, `qwen3-coder:30b`, …),
or set `llm.tool_mode: json` to use the JSON-in-text fallback (less reliable). `docker logs ollama --tail 20` shows the actual message.

**Japanese / CJK input method (Mozc, fcitx5, ibus) does not work in the dedicated window**

The Qt shipped by pip (`pywebview[qt]`) does not include the fcitx input-method plugin, so the IME never activates
inside a Qt-backed window, even though it works fine in Firefox or Chromium. Switch to the GTK backend, which uses the
system WebKit2GTK and therefore the system input method:

```bash
bash setup_linux_window.sh
```

The script installs `python3-gi` + WebKit2GTK bindings via apt, recreates `.venv` with `--system-site-packages`
(re-installing requirements) and removes the pip Qt packages. `app.py` prefers the GTK backend whenever it is available.

**The dedicated window does not open and the default browser is used instead**

Check `~/.apercode/app.log`. If it says `Could not load the Qt platform plugin "xcb"`, install `libxcb-cursor0`
(Debian/Ubuntu: `sudo apt install -y libxcb-cursor0`).

**Keeping the agent inside the working directory**

Since 1.1.4, any tool call that points outside the session's working directory (absolute paths, `~/…`, `../`,
or `cd` / absolute paths inside `run_command`) asks for approval even for read-only tools. Approve once, or choose
"always allow for this session". `permissions.outside_project` can be set to `ask` / `allow` / `deny`.
Paths outside `workspace.root` are always rejected.

**The model stops responding right after "saved to memory"**

Fixed in 1.1.7: the post-turn reflection (an extra chat call plus loading the embedding model) now runs in the
background with a timeout, the transcript is capped, the embedding model is released quickly (`keep_alive: 2m`),
the chat model is kept resident (`llm.keep_alive`), and transient Ollama outages are retried. If it still happens,
lower `llm.num_ctx`, clear `memory.embed_model`, or set `memory.reflect: off` (VRAM pressure).

**A new session in a new working directory still works in the previous directory**

Fixed in 1.1.3. Two causes: (1) a server started earlier with `python3 main.py` was still running and `app.py`
attached to it; the launcher now compares the running server's version with the code and restarts it when they
differ. (2) experience memories mentioning other projects' paths pulled the model away; the system prompt now
states the current working directory first and tells the model not to follow paths from memories.

**The model dropdown is rendered white-on-white in the GTK window**

WebKitGTK draws `<select>` with the OS theme. Fixed in 1.1.2 (custom-drawn select, dark colors); update `index.html`.

**`web_search` fails with HTTP 403**

The search site is blocking non-browser clients. Install the `ddgs` package (included in requirements.txt since
v1.2.1.1; for an existing venv run `.venv/bin/pip install ddgs`) — it connects with browser-like TLS and is tried
first. If it still fails, the tool falls back to Bing → Brave → Mojeek → Stack Exchange / GitHub / Wikipedia APIs;
the first line of the result names the engine that was used. If every engine fails, the error lists each engine's
result so you can tell whether your network is blocking them. A self-hosted SearXNG can be set via `web.searxng_url`.

**Pressing Enter to confirm an IME conversion sends the message**

Fixed: Enter is ignored while a composition is in progress. Make sure you are on the latest `index.html`.

## Configuration

Settings are merged in this order (later wins):
1. `config.example.yaml` (bundled defaults)
2. `config.yaml` (project root, git-ignored)
3. `~/.apercode/config.yaml` (per-user overrides)

Key settings: `llm.model`, `llm.num_ctx`, `workspace.root` (the only directory tree the agent may touch),
`permissions.*`, `compaction.*`, `subagents.*`, `memory.*`, `browser.*`, `repl.*`, `process.*`, `mt5.*`, `brief.clarify`, `peer.*`, `peers`.

User data (sessions, memory DB, logs) lives in `~/.apercode/`.

## Layout

```
app.py                  desktop launcher
main.py                 server only
codeagent/agent.py      agent loop, compaction, sub-agents, memory recall/reflection
codeagent/llm.py        Ollama / Anthropic client
codeagent/memory/       experience memory store
codeagent/tools/        fs, shell, mt5, subagent, memory, web tools
codeagent/server/       FastAPI + WebSocket, static UI, peer API (peer_api.py)
```

## License

MIT
