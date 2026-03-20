# Airia On-Call

**Autonomous on-call engineering agent. Bug comes in. Four agents fix it in parallel. You approve in one tap.**

Built for the Airia AI Agent Challenge — Track 2: Active Agents.

---

## What it does

When a bug ticket arrives, Airia On-Call launches four specialized agents simultaneously:

| Agent | Role |
|---|---|
| **Triage Agent** | Maps acceptance criteria, labels priority, identifies affected components |
| **Code Agent** | Searches the codebase, writes and commits the fix to a new branch |
| **Test Agent** | Generates regression tests, runs the full suite, verifies all pass |
| **PR Agent** | Drafts a complete pull request description |

All four run in parallel — not sequentially. When they finish, a structured Slack briefing appears with an **Approve & Open PR** button. One tap opens the PR on GitHub. The full loop from bug report to merged-ready PR, with a single human decision.

---

## Demo flow

```
/oncall <bug description>
      ↓
Four agents launch in parallel (Triage · Code · Test · PR)
      ↓
Slack briefing: root cause · fix · test results · CI status
      ↓
[ Approve & Open PR ]  [ Request Changes ]
      ↓
PR opens on GitHub  ·  Jira ticket closes with PR linked
```

---

## Architecture

```
main.py  (FastAPI — Slack slash command + interactive callbacks)
│
├── workflow/oncall_pipeline.py     ← parallel agent orchestrator
│   └── asyncio.gather(triage, code, test, pr)
│
├── agents/definitions.py           ← SubagentTool definitions
│
├── integrations/
│   ├── slack_bot.py                ← Block Kit briefing + HITL buttons
│   ├── github_client.py            ← branch · commit · PR
│   ├── jira_client.py              ← ticket ingestion + status updates
│   └── cicd_monitor.py             ← CI failure log ingestion
│
├── documents/doc_generator.py      ← PR desc · Slack briefing · incident report
│
└── tools/builtin/                  ← file ops · shell · search · test runner · MCP
```

The agent engine (under `agent/`, `tools/`, `context/`, `safety/`) provides:
- Streaming agentic loop with tool use
- Session persistence (save / resume mid-workflow)
- Context compression for long runs
- Safety layer with configurable approval policies
- MCP protocol client for extensible tool connections
- Lifecycle hooks (before/after agent, before/after tool, on_error)

---

## Integrations

- **Slack** — slash command trigger (`/oncall`), Block Kit briefings, interactive Approve / Request Changes buttons
- **GitHub** — branch creation, file commits, pull request creation with labels
- **Jira** — ticket ingestion, status updates, PR linking on close
- **CI/CD** — GitHub Actions log ingestion, failure analysis
- **Airia Platform** — document generation API for incident reports

---

## Setup

### 1. Clone and install

```bash
git clone https://github.com/PranavPipariya/airia-oncall.git
cd airia-oncall
pip install -r requirements.txt
```

### 2. Configure

```bash
cp .env.example .env
# Fill in your keys (see .env.example for details)
```

Required:
- `API_KEY` + `BASE_URL` — LLM provider (OpenRouter recommended)
- `GITHUB_TOKEN` + `GITHUB_REPO` — repo to open PRs against
- `SLACK_BOT_TOKEN` + `SLACK_CHANNEL` — Slack bot with `chat:write`, `chat:write.public`, `commands` scopes

### 3. Expose locally with ngrok

```bash
ngrok http 8000
```

In your Slack app settings:
- **Slash Commands** → Request URL: `https://<your-ngrok>.ngrok.io/webhook/slack/command`
- **Interactivity & Shortcuts** → Request URL: `https://<your-ngrok>.ngrok.io/webhook/slack/actions`

### 4. Start

```bash
uvicorn main:app --port 8000
```

### 5. Trigger

In Slack, type:
```
/oncall <describe the bug>
```

---

## Slack app scopes

```
chat:write
chat:write.public
commands
```

---

## Documents generated per run

Every pipeline run produces three documents automatically:

1. **PR Description** — problem, root cause, fix summary, files changed, test results
2. **Slack Briefing** — human-readable incident summary for HITL approval
3. **Incident Report** — full root cause analysis, fix explanation, CI status

---

## Built with

- Python 3.9+
- FastAPI — webhook server
- slack-sdk — Slack Block Kit + interactive components
- PyGithub — GitHub REST API
- anthropic / openrouter — LLM backbone
- asyncio — parallel agent execution
- Airia Platform API — document generation
