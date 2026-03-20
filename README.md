# Anton

> **Your autonomous on-call engineer.**
> A bug lands. Four AI agents fix it in parallel. You approve in one tap. PR is open.

Anton is a fully autonomous on-call agent built on a custom multi-agent orchestration engine. When a bug ticket arrives — via Slack slash command, Jira webhook, or direct API trigger — Anton spins up four specialized subagents simultaneously, each with its own isolated context, toolset, and goal. They work in parallel, not in sequence. When all four finish, a structured Slack briefing appears in your channel with a single **Approve & Open PR** button. One tap. PR on GitHub. Ticket closed. Done.

No dashboards. No context switching. No 2am pages that drag on for hours.

---

## The Problem Anton Solves

Modern engineering teams are drowning in operational toil. A production bug comes in at 2am. An on-call engineer wakes up, spends 20 minutes reading logs, 30 minutes finding the bug, 20 minutes writing a fix, 15 minutes writing tests, 10 minutes writing a PR description. That's 95 minutes of mechanical work that a machine should be doing.

Anton does that 95 minutes in under 3 minutes. The engineer wakes up to a Slack message, reads a one-paragraph summary, taps Approve, and goes back to sleep.

---

## How It Works

```
User types in Slack:
  /oncall todo complete bug — calling complete twice un-completes the todo

                        │
                        ▼
              ┌─────────────────────┐
              │   Anton Orchestrator │
              │   (FastAPI + asyncio)│
              └─────────┬───────────┘
                        │
          ┌─────────────┼─────────────────┐
          │             │                 │              │
          ▼             ▼                 ▼              ▼
   ┌────────────┐ ┌──────────────┐ ┌───────────┐ ┌──────────┐
   │   TRIAGE   │ │     CODE     │ │   TEST    │ │    PR    │
   │   AGENT    │ │    AGENT     │ │   AGENT   │ │  AGENT   │
   │            │ │              │ │           │ │          │
   │ • Priority │ │ • Read repo  │ │ • Read    │ │ • Draft  │
   │ • Component│ │ • Find bug   │ │   tests   │ │   full   │
   │ • Criteria │ │ • Write fix  │ │ • Write   │ │   PR     │
   │ • Labels   │ │ • Output     │ │   new     │ │   desc   │
   │            │ │   file diff  │ │   cases   │ │          │
   │            │ │              │ │ • Run     │ │          │
   │            │ │              │ │   suite   │ │          │
   └────────────┘ └──────────────┘ └───────────┘ └──────────┘
          │             │                 │              │
          └─────────────┴─────────────────┴──────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │  Document Generator  │
              │  • PR description    │
              │  • Slack briefing    │
              │  • Incident report   │
              └─────────┬───────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │        Slack Briefing          │
        │  🚨 Anton — Incident Ready    │
        │  Ticket: BUG-42 · P1 Critical │
        │  Root cause: toggle vs assign  │
        │  Fix: todos/service.py:L47    │
        │  Tests: 5 new · all passing ✅ │
        │  CI: all checks green ✅       │
        │                               │
        │  [ ✅ Approve & Open PR ]      │
        │  [ ✏️  Request Changes ]       │
        └───────────────┬───────────────┘
                        │
                Human taps Approve
                        │
                        ▼
        ┌───────────────────────────────┐
        │  • PR opened on GitHub        │
        │  • Jira ticket closed + linked│
        │  • Slack message updated      │
        │  • Incident report committed  │
        └───────────────────────────────┘
```

---

## The Four Agents

Each agent runs in a completely isolated subagent context. They share the same repository working directory but have separate conversation histories, separate tool access, and separate timeouts. The orchestrator launches all four with `asyncio.gather()` and collects their outputs when all complete.

### Triage Agent
**Purpose:** Pure reasoning. No file access.

Reads the bug description and outputs a structured JSON object containing priority classification (P1–P4), affected component name, a list of acceptance criteria the fix must satisfy, and a list of edge cases to test. This output is injected into the Code Agent and Test Agent goals to give them context without them needing to re-read the ticket.

Allowed tools: none. This is intentional — triage is about reasoning over the ticket text, not exploring the codebase. Keeping it tool-free makes it deterministic and fast (under 60 seconds).

```json
{
  "priority": "P1 Critical",
  "component": "todo-service",
  "affected_files_hint": ["todos/service.py"],
  "acceptance_criteria": [
    "complete() always sets completed=True, never toggles",
    "Calling complete() twice on the same todo is idempotent"
  ],
  "edge_cases": ["complete already-completed todo", "complete non-existent todo"]
}
```

### Code Agent
**Purpose:** Find the bug. Write the fix. Nothing more.

Has read access to the full repository. Searches files, reads the relevant module, identifies the exact line(s) causing the bug, and outputs the corrected file content in a structured format the orchestrator can parse and commit. Strictly instructed not to refactor unrelated code — only touch what is broken.

Allowed tools: `read_file`, `grep`, `glob`, `list_dir`

Output format:
```
CHANGED_FILES:
- todos/service.py

FIX_EXPLANATION:
complete() used `not todo.completed` (toggle) instead of `todo.completed = True` (set).

FILE: todos/service.py
```python
<full corrected file content>
```
```

### Test Agent
**Purpose:** Write regression tests. Run the suite. Prove the fix works.

Has read and write access to test files, plus the ability to execute the test runner. Reads existing tests to understand the test style and fixture setup, generates new test cases covering the bug scenario and all edge cases from the triage output, writes them into the test file, runs the full suite, and reports pass/fail. If tests fail, it attempts one self-correction cycle before reporting.

Allowed tools: `read_file`, `write_file`, `run_tests`, `grep`, `glob`

The Test Agent does not see the Code Agent's output — it derives the fix independently from the ticket description. This acts as a form of cross-validation: if both agents arrive at the same conclusion, confidence in the fix is high.

### PR Agent
**Purpose:** Write the pull request description.

Pure text synthesis. Given the ticket summary, priority, and description, drafts a complete PR description with Problem / Root Cause / Solution / Files Changed / Tests / How to Verify sections. Writes the kind of PR description a senior engineer would write after fully understanding a change — not a template.

Allowed tools: none.

---

## Architecture Deep Dive

### Project Structure

```
anton/
│
├── main.py                          # FastAPI server — all webhook endpoints
│
├── workflow/
│   └── oncall_pipeline.py           # Core orchestrator — parallel agent execution
│
├── agents/
│   └── definitions.py               # SubagentDefinition objects for all 4 agents
│
├── integrations/
│   ├── slack_bot.py                 # Block Kit message builder + HITL callbacks
│   ├── github_client.py             # Branch creation, file commits, PR creation
│   ├── jira_client.py               # Ticket ingestion + status updates
│   └── cicd_monitor.py              # CI failure log ingestion
│
├── documents/
│   └── doc_generator.py             # Generates PR desc, Slack briefing, incident report
│
├── agent/                           # Core agentic loop engine
│   ├── orchestrator.py              # Streaming agent loop with tool calling
│   ├── session_manager.py           # Session lifecycle management
│   ├── session_persistence.py       # Save/resume sessions to disk
│   └── event_types.py               # Typed event system for agent lifecycle
│
├── tools/
│   ├── specialized_agents.py        # SubagentTool — spawns isolated child agents
│   ├── tool_registry.py             # Dynamic tool registration
│   ├── plugin_loader.py             # Load tools from external plugins
│   ├── mcp/                         # Model Context Protocol client
│   │   ├── mcp_client.py
│   │   ├── mcp_connection_manager.py
│   │   └── mcp_tool_adapter.py
│   └── builtin/
│       ├── file_reader.py           # Read files from the working directory
│       ├── file_writer.py           # Write files with path safety checks
│       ├── file_editor.py           # Surgical line-level edits
│       ├── shell_executor.py        # Run shell commands (sandboxed)
│       ├── test_executor.py         # Run pytest and parse results
│       ├── test_generator.py        # Generate test stubs
│       ├── text_search.py           # Grep over repository
│       ├── pattern_matcher.py       # Glob file patterns
│       ├── directory_listing.py     # List directory contents
│       ├── github_tools.py          # GitHub API tool wrappers
│       ├── jira_tool.py             # Jira API tool wrappers
│       ├── slack_tool.py            # Slack messaging tools
│       ├── web_fetcher.py           # HTTP fetch for external context
│       ├── web_searcher.py          # DuckDuckGo search
│       └── persistent_memory.py     # Cross-session agent memory
│
├── context/
│   ├── conversation_manager.py      # Token budget management
│   ├── context_compressor.py        # Compress long conversations
│   └── infinite_loop_detector.py    # Detect and break agent loops
│
├── safety/
│   └── permission_manager.py        # Approval policies: auto / ask / never
│
├── hooks/
│   └── lifecycle_hooks.py           # before_agent / after_tool / on_error hooks
│
├── config/
│   ├── configuration.py             # Typed config dataclasses
│   └── config_loader.py             # TOML config file loader
│
├── context/
├── ui/
│   └── terminal_interface.py        # Rich terminal UI for CLI mode
│
├── demo/
│   ├── trigger.py                   # Fire a demo webhook without Slack
│   └── sample_repo/                 # Buggy todo service for demo
│       ├── main.py                  # FastAPI todo app
│       ├── todos/
│       │   ├── models.py
│       │   └── service.py           # ← bug lives here
│       └── tests/
│           └── test_service.py
│
├── .env.example                     # All environment variables documented
└── requirements.txt
```

### The Agent Engine

Anton is built on a custom agentic loop engine — not LangChain, not AutoGen, not CrewAI. Every layer is purpose-built:

**Orchestrator (`agent/orchestrator.py`)**
The core loop: send message to LLM → parse tool calls → execute tools → feed results back → repeat until the agent signals it's done or the turn limit is hit. Streaming-first: tokens arrive and are processed as they stream, tool calls are dispatched as soon as the LLM finishes the tool call block. No waiting for the full response before acting.

**SubagentTool (`tools/specialized_agents.py`)**
The mechanism for parallel agents. Each call to `SubagentTool.execute()` creates a brand-new agent instance with its own config, its own allowed tool set, and its own conversation history. The parent orchestrator gets back a single string — the child agent's final response. The child's internal conversation is invisible to the parent, preventing context bleed between agents.

**Context Management (`context/`)**
Long-running agents accumulate large conversation histories. The `ContextCompressor` summarises earlier turns when the token budget approaches its limit, preserving the essential facts while discarding the verbose intermediate steps. The `InfiniteLoopDetector` tracks tool call patterns and breaks cycles if an agent is calling the same tool with the same arguments repeatedly.

**Safety Layer (`safety/permission_manager.py`)**
Three modes: `auto` (all tools execute without confirmation), `ask` (dangerous tools require approval), `never` (shell commands blocked entirely). The on-call pipeline runs in `auto` mode by default — agents need to read and write files without interruption. The shell executor is scoped to the repository working directory.

**MCP Client (`tools/mcp/`)**
Full Model Context Protocol client implementation. Anton can connect to any MCP server and expose its tools to agents, making the tool set extensible without modifying core code. Add a database MCP server and agents can query your schema. Add a cloud provider MCP server and agents can read infrastructure state.

### Parallel Execution

The orchestrator runs all four agents with a single `asyncio.gather()` call:

```python
triage_raw, code_raw, test_raw, pr_raw = await asyncio.gather(
    _run_subagent(TRIAGE_AGENT, triage_goal, repo_cwd),
    _run_subagent(CODE_AGENT,   code_goal,   repo_cwd),
    _run_subagent(TEST_AGENT,   test_goal,   repo_cwd),
    _run_subagent(PR_AGENT,     pr_goal,     repo_cwd),
)
```

All four agents make their LLM API calls, tool calls, and file reads concurrently. On a typical bug, this means 4 agents complete in the time it would take 1 agent to finish sequentially. The Code Agent's execution time dominates (it reads files, reasons about the codebase, writes a fix), so the other three agents finish first and wait. Total pipeline time: typically 60–120 seconds depending on repository size and bug complexity.

### HITL via Slack Block Kit

The Slack briefing is built with Block Kit — Slack's structured layout system. This is not a text message; it's a rich interactive card with:

- Header block with incident title
- Section blocks with ticket key, priority, component, and branch name
- Section blocks for root cause and fix summary
- File list of changed files
- Test count and CI status
- **Actions block** with Approve and Request Changes buttons

Each button carries a `value` field encoded as `"approve|{run_id}"` or `"reject|{run_id}"`. When a button is pressed, Slack sends a POST to `/webhook/slack/actions` with the full interaction payload. The server parses the `run_id`, looks up the in-memory `PipelineRun` object, and dispatches the approval or rejection handler as a background task. The HTTP response is returned immediately (Slack requires a response within 3 seconds) and the actual work happens asynchronously.

On approval, the pipeline:
1. Creates a new branch on GitHub
2. Commits each changed file from the Code Agent's output
3. Commits the updated test file from the Test Agent's output
4. Opens a pull request with the PR Agent's description
5. Updates the Slack message to show the PR link
6. Links the PR to the Jira ticket and closes it

On rejection, the pipeline:
1. Updates the Slack message to show the feedback
2. Re-runs the Code Agent with the human's feedback prepended to the goal
3. Generates a revised fix

### Document Generation

Every pipeline run produces three documents automatically:

**PR Description**
The full GitHub pull request body. Written by the PR Agent but structured by the document generator with standardised sections. Committed to the PR body on creation.

**Slack Briefing**
The structured Block Kit payload. Designed to be scannable in under 30 seconds — a human on-call engineer should be able to understand the bug, the fix, and the risk level without reading a single line of code.

**Incident Report**
A markdown document saved to `.oncall_runs/incident_{ticket_key}_{run_id}.md` in the repository. Contains the full pipeline run metadata: ticket details, triage output, fix explanation, test results, CI status, agent execution times, and the final decision. Useful for post-mortems and audit trails.

---

## Integrations

### Slack
- **Trigger:** `/oncall <bug description>` slash command
- **Briefing:** Block Kit interactive message with Approve / Request Changes buttons
- **Callbacks:** Interactive component webhook at `/webhook/slack/actions`
- **Updates:** Message is updated in-place after approval or rejection (no duplicate messages)
- **Required scopes:** `chat:write`, `chat:write.public`, `commands`

### GitHub
- **Branch creation:** `fix/{ticket_key}-{run_id}` naming convention
- **File commits:** Each changed file committed individually with descriptive commit messages
- **PR creation:** Full description, labels (`bug`, `anton`), linked to ticket key
- **Fallback:** If a PR already exists for the branch, returns the existing PR URL

### Jira
- **Ticket ingestion:** Reads ticket summary, description, priority, labels, components via Jira REST API v3
- **Status updates:** Transitions ticket to "In Progress" when pipeline starts
- **PR linking:** Posts a comment with the PR URL and transitions ticket to "Done" on approval
- **Mock fallback:** If Jira credentials are not configured, uses a rich demo ticket automatically

### CI/CD
- **Log ingestion:** Reads CI failure logs from the repository to give the Code Agent additional context
- **Status reporting:** CI pass/fail status shown in the Slack briefing
- **GitHub Actions compatible:** Parses standard pytest output from CI logs

---

## Setup

### Prerequisites
- Python 3.9+
- A Slack workspace where you can install apps
- A GitHub account with a repository to test against
- An LLM API key (OpenRouter recommended — gives access to Claude, GPT-4o, Llama, and more)
- ngrok (or any tunnel) to expose your local server to Slack's callbacks

### 1. Clone and install

```bash
git clone https://github.com/PranavPipariya/anton.git
cd anton
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
```

Open `.env` and fill in:

```bash
# LLM provider — OpenRouter gives access to Claude, GPT-4o, Llama
API_KEY=sk-or-v1-...
BASE_URL=https://openrouter.ai/api/v1

# GitHub — the repo Anton will open PRs against
GITHUB_TOKEN=ghp_...
GITHUB_REPO=yourname/your-repo

# Slack bot token and channel name
SLACK_BOT_TOKEN=xoxb-...
SLACK_CHANNEL=your-channel-name

# Jira (optional — demo mode used if not set)
JIRA_BASE_URL=https://your-org.atlassian.net
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=...
```

### 3. Create your Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → From scratch
2. Name it `Anton`, pick your workspace
3. **OAuth & Permissions** → Bot Token Scopes → add:
   - `chat:write`
   - `chat:write.public`
   - `commands`
4. **Install to Workspace** → copy the **Bot User OAuth Token** → paste as `SLACK_BOT_TOKEN`
5. Create a channel in your workspace (e.g. `#anton`) and invite the bot: `/invite @Anton`

### 4. Expose local server with ngrok

```bash
ngrok http 8000
```

Copy the `https://` URL (e.g. `https://abc123.ngrok-free.app`).

In your Slack app settings:

**Slash Commands** → Create New Command:
- Command: `/oncall`
- Request URL: `https://abc123.ngrok-free.app/webhook/slack/command`
- Short description: `Trigger Anton on a bug`

**Interactivity & Shortcuts** → turn on → Request URL:
```
https://abc123.ngrok-free.app/webhook/slack/actions
```

### 5. Start the server

```bash
uvicorn main:app --port 8000
```

You should see:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🚨  Anton — Ready
  Repo CWD : ./demo/sample_repo
  Jira     : mock
  Slack    : live
  GitHub   : live
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 6. Run the demo

In your Slack channel:

```
/oncall todo complete bug — calling complete() twice un-completes the todo
```

Anton will:
1. Confirm activation in the channel immediately
2. Launch four agents in parallel
3. Post a structured briefing in 60–120 seconds
4. Wait for your tap on **Approve & Open PR**
5. Open the PR on GitHub and post the link back to Slack

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/webhook/jira` | Jira issue created/updated webhook |
| `POST` | `/webhook/slack/command` | Slack `/oncall` slash command |
| `POST` | `/webhook/slack/actions` | Slack interactive button callbacks |
| `GET` | `/health` | Health check — returns service status and active run count |
| `GET` | `/runs` | List all active pipeline runs |
| `GET` | `/runs/{run_id}` | Get detailed status of a specific run |
| `POST` | `/runs/{run_id}/approve` | Programmatically approve a run (for testing) |
| `POST` | `/runs/latest/approve` | Approve the most recent run |

---

## Demo Repository

`demo/sample_repo/` contains a minimal Python todo service with a real, intentional bug:

**The bug:** `TodoService.complete()` uses `not todo.completed` (a toggle) instead of `todo.completed = True` (a set). The first call to `complete()` works correctly. The second call on the same todo reverts it back to incomplete — silently, with no error.

This is a real class of bug that exists in production codebases. The failing tests in `tests/test_service.py` precisely define the expected behaviour:

```python
def test_complete_is_idempotent(svc):
    """Completing an already-completed todo should keep it completed."""
    todo = svc.create("Deploy to prod")
    svc.complete(todo.id)   # False → True ✓
    svc.complete(todo.id)   # BUG: True → False ✗
    assert svc.get(todo.id).completed is True  # fails
```

Anton's Code Agent finds this in under 90 seconds, writes the one-line fix, and the Test Agent independently verifies it with regression tests.

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `API_KEY` | ✅ | LLM provider API key |
| `BASE_URL` | ✅ | LLM provider base URL (omit for OpenAI direct) |
| `GITHUB_TOKEN` | ✅ | GitHub personal access token with repo scope |
| `GITHUB_REPO` | ✅ | Target repository in `owner/repo` format |
| `SLACK_BOT_TOKEN` | ✅ | Slack bot token (`xoxb-...`) |
| `SLACK_CHANNEL` | ✅ | Slack channel name (without `#`) |
| `JIRA_BASE_URL` | ❌ | Jira instance URL — demo mode if not set |
| `JIRA_EMAIL` | ❌ | Jira account email |
| `JIRA_API_TOKEN` | ❌ | Jira API token |
| `REPO_CWD` | ❌ | Path to local repo clone — defaults to `demo/sample_repo` |

---

## Built With

| Library | Purpose |
|---------|---------|
| `FastAPI` | Webhook server and REST API |
| `uvicorn` | ASGI server |
| `slack-sdk` | Slack Block Kit messages and interactive components |
| `PyGithub` | GitHub REST API — branches, commits, pull requests |
| `openai` | LLM API client (works with OpenRouter, OpenAI, any OpenAI-compatible API) |
| `asyncio` | Concurrent agent execution |
| `pydantic` | Typed data models |
| `tiktoken` | Token counting for context budget management |
| `rich` | Terminal output formatting |
| `python-dotenv` | Environment variable loading |
| `duckduckgo-search` | Web search tool for agents |
| `httpx` | Async HTTP client |

---

## License

MIT
