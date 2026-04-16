# AI Research Agent

An automated multi-AI research orchestrator that runs in Claude Code. You type a topic, and it queries **Claude.ai**, **Gemini**, **ChatGPT**, and **Grok** in parallel via browser automation, then synthesizes a single cross-source report.

All four platforms are driven through a real Chrome window using the Claude in Chrome extension — no API keys required. You use your own paid subscriptions, in their full Research / Deep Research / Expert modes.

## What it does

- Refines your casual question into a structured research brief.
- Submits that brief to all 4 AI platforms in parallel browser tabs (Claude Research, Gemini Deep Research, ChatGPT Deep Research, Grok Expert).
- Polls each tab until completion (sessions can run 30+ minutes — that's normal for Deep Research modes).
- Extracts each response, including source URLs, into local markdown files.
- Synthesizes a single themed report comparing consensus, disagreements, and unique insights across all 4 sources.
- Self-evaluates the result and tracks scores over time in a learning ledger.
- Optionally mirrors session status to a Notion dashboard.

## Requirements

- **Claude Code** — the Anthropic CLI ([install instructions](https://docs.claude.com/claude-code)).
- **Google Chrome** + the **Claude in Chrome** extension (for browser automation).
- A dedicated Chrome profile signed into all four platforms.
- Paid accounts on each platform (the deep research modes are paywalled):
  - Claude.ai — Pro or higher
  - Gemini — Advanced
  - ChatGPT — Plus or higher
  - Grok — grok.com account with Expert mode access
- (Optional) Notion + the Notion MCP, if you want a live dashboard.

**Supported operating systems:** Windows 10/11, macOS, Linux. The setup flow asks which OS you're on and adjusts path examples accordingly. All shell commands are bash-compatible (Windows users run Claude Code in Git Bash, which is installed automatically with Claude Code).

## Install

1. Clone this repo to wherever you want it to live:
   ```
   git clone <this-repo-url>
   cd ai-research-agent
   ```
2. Open the folder in Claude Code (`claude` from inside the directory).
3. In your first chat, just say **"setup"** (or "get started", "how do I begin", etc.). Claude will offer a numbered list of setup tasks and do whichever ones you pick — folder creation, Chrome setup walkthrough, account checklist, optional Notion integration.

That's the entire install. The agent's own setup flow is the install guide.

## Use

Once setup is done, start a research session by typing:

```
Research: <your topic>
```

Examples:
- `Research: how octopus cognition compares to mammalian cognition`
- `Research the current state of solid-state battery commercialization`

Other commands:
- **`Resume research`** — pick up an interrupted or partially-failed session.
- **`Review research performance`** — see scoring trends across your past sessions (needs 3+ sessions).
- Saying "stop" or "cancel" mid-run cancels cleanly.

Each session writes everything to `research/<session-id>/` — research brief, raw responses from each platform, sources, the final synthesized report, and a self-evaluation. The `report.md` file is the main deliverable.

## Configuration

Local config lives in `config.local.json` (gitignored). A template is at `config.example.json`. The setup flow creates the local copy for you on first run.

Keys:
- `chrome_profile_name` — the Chrome profile signed into the 4 platforms.
- `notion.enabled` — `true` to mirror sessions to Notion, `false` to run purely locally.
- `notion.database_id`, `notion.data_source_id`, `notion.dashboard_page_id` — only used when Notion is enabled. The setup flow walks you through getting these.

## How it works

The full orchestration logic lives in [CLAUDE.md](CLAUDE.md). When you open the project in Claude Code, that file becomes Claude's instructions — it defines the triggers, the workflow steps, the per-platform extraction methods, error handling, and the self-evaluation rubric. If you want to change the agent's behavior, edit `CLAUDE.md`.

The shared system prompt and per-platform prompt templates live in `prompts/`. The report structure template lives in `templates/report.md`. Edit those to change what the AIs are asked to produce.

## What gets shared vs. what stays local

The repo ships with the orchestration logic, prompt templates, report template, and evaluation schema. The following are gitignored and stay on your machine:
- Your `config.local.json` (Notion IDs, profile name)
- Your `research/` sessions
- Your `learning/ledger.json` (your accumulated session scores)
- Browser download artifacts in `Downloads/`

You can safely share this repo without leaking your data.

## Notes

- **Deep Research modes are slow.** ChatGPT can take 20–30 minutes, Gemini can take an hour or more. This is normal.
- **AI sources are unverified.** All four platforms occasionally hallucinate URLs. The synthesized report flags source lists as "cited by platforms, not independently verified."
- **Each session is independent.** Platforms never see each other's output — synthesis happens locally in Claude Code.

## License

[MIT](LICENSE) — use it, fork it, modify it, ship it.
