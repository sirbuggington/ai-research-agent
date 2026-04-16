# First-Time Setup

**Load this doc when the user triggers setup.** Trigger phrases: "setup", "set up", "get started", "let's get started", "how do I start", "how do I begin", "first time setup", "install", "initialize", or anything clearly meaning the same thing.

**Use the `AskUserQuestion` tool to run a short questionnaire — NOT a text list.** This pops up a native multiple-choice / yes-no UI in Claude Code so the user can click answers instead of typing them.

## Step A: Run the questionnaire

Call `AskUserQuestion` with these two questions in a single call:

1. **Question:** `"What's your operating system?"`
   - header: `OS`
   - multiSelect: false
   - options:
     - label: `Windows`, description: `"Windows 10 or 11 (Claude Code runs in bash)"`
     - label: `macOS`, description: `"Apple Silicon or Intel Mac"`
     - label: `Linux`, description: `"Any modern Linux distribution"`

2. **Question:** `"Which setup tasks should I handle now?"`
   - header: `Tasks`
   - multiSelect: true
   - options:
     - label: `Create folders + config`, description: `"Scaffold the folder structure and copy config.example.json to config.local.json (Recommended)"`
     - label: `Walk through Chrome setup`, description: `"Install the Claude in Chrome extension, create a dedicated profile, set the download folder, connect the MCP"`
     - label: `Account checklist`, description: `"Which AI subscriptions you need and what to verify before the first run"`
     - label: `Notion integration`, description: `"Optional dashboard — I'll walk you through creating the database and pasting the IDs into your config"`

Do not ask anything else before running this. Do not write a preamble paragraph — call the tool immediately.

## Step B: Act on the answers

Run ONLY the tasks the user selected. Each task block below is independent. Keep prose minimal — be concise.

**Store the OS answer.** Use it to format every example path you show the user. The actual project root is your current working directory — use it when possible. Only fall back to the generic examples below if the cwd isn't obvious.

Path examples by OS (replace `<project>` with the actual project path when you know it):
- **Windows:** `C:\path\to\ai-research-agent\Downloads\` — backslashes for display to the user, forward slashes inside bash commands
- **macOS:** `/Users/<you>/path/to/ai-research-agent/Downloads/`
- **Linux:** `/home/<you>/path/to/ai-research-agent/Downloads/`

### Task: Create folders + config

1. Check which of these exist at the project root and create only what's missing: `prompts/`, `templates/`, `learning/`, `research/`, `Downloads/`.
2. If `config.local.json` is missing, copy `config.example.json` to `config.local.json`.
3. If `learning/ledger.json` is missing, create it: `{"ledger_version": "1.0", "sessions": [], "promotions": []}`.
4. If `learning/adjustments.json` is missing, create it: `{"adjustments_version": "1.0", "mode": "recommendation_only", "active_adjustment": null, "history": [], "pending_recommendations": []}`.
5. Tell the user one line: "Folders ready. Open `config.local.json` and set `chrome_profile_name`."

### Task: Walk through Chrome setup

Show these steps numbered, nothing else:

1. Install the **Claude in Chrome** extension from the Chrome Web Store.
2. In Claude Code, run `/mcp` to confirm the Claude in Chrome MCP server is connected. If it isn't, follow the prompts to add it — the agent cannot drive the browser without it.
3. Create a dedicated Chrome profile for research (recommended name: `AI Research`) via Chrome menu -> profile picker -> Add.
4. In that profile's Chrome settings -> Downloads -> Location, set the download folder to the project's `Downloads/` directory. Show the OS-specific path.
5. **Heads up:** Chrome's download location is per-profile, so every download in this profile will land in the project folder. That's why a dedicated profile is strongly recommended.
6. Put the profile name into `config.local.json` -> `chrome_profile_name`.

### Task: Account checklist

Show this list and nothing else:

1. **Claude.ai** — Pro or higher (Research mode is paywalled).
2. **Gemini** — Google account with Gemini Advanced (Deep Research requires Advanced).
3. **ChatGPT** — Plus or higher (Deep Research is paywalled).
4. **Grok** — grok.com account with Expert mode access.

Then: "Sign into all four in your research Chrome profile. Open each site once to verify no login prompt appears."

### Task: Notion integration

1. Install or connect the **Notion MCP** in Claude Code (run `/mcp` and add it). Grant it access to a workspace you control.
2. In that workspace, create a new database called **"AI Research Sessions"** with these exact properties: `Topic` (title), `Status` (select), `Claude` (select), `Gemini` (select), `ChatGPT` (select), `Grok` (select), `Platforms` (text), `Tags` (multi-select), `Started` (date), `Completed` (date), `Session ID` (text).
3. Status select options: `Running`, `Synthesizing`, `Completed`, `Completed with Errors`, `Failed`, `Cancelled`. Each platform select needs: `Pending`, `Running`, `Done`, `Failed`, `Timed Out`, `Skipped`, `Cancelled`.
4. Create a separate page called something like **"AI Research Dashboard"** — session archive toggles will be appended here after each run.
5. Collect three IDs:
   - **Database ID** — the 32-char hex string in the database URL (before `?`).
   - **Data source ID** — after connecting the Notion MCP, ask Claude to call `notion-fetch` on the database URL and read the returned `data_source_id`.
   - **Dashboard page ID** — same format, from the dashboard page URL.
6. Edit `config.local.json`: set `notion.enabled` to `true` and paste the three IDs into `database_id`, `data_source_id`, `dashboard_page_id`.

## Step C: Wrap up

Finish with one short line: "Setup tasks done. To start a research session, type `Research: <your topic>`."

Do NOT auto-launch a research session after setup. Wait for the user to trigger one.
