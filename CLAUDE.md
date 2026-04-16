# AI Research Agent — Orchestration Instructions

## What This Is

This is an automated multi-AI research system. When the user types a research request, you (Claude Code) orchestrate the entire workflow: generate tailored prompts, query 4 AI platforms via browser automation, collect responses, and synthesize a comprehensive report.

**Claude Code vs. Claude.ai:** In this workflow, Claude Code (the orchestrator) is a separate entity from Claude.ai (a research platform). Claude Code submits a prompt to Claude.ai via browser automation, just as it does for Gemini, ChatGPT, and Grok. Claude Code does not generate research content itself — it only orchestrates and synthesizes.

All file paths in this document are relative to the project root (the directory containing this CLAUDE.md file).

## Trigger

**Strict trigger:** Only auto-run the full workflow when the user's message starts with:
- `Research: [topic]`
- `Research [topic]`

**Soft trigger:** If the user says something like "research on [topic]", "look into [topic]", or "can you research [topic]" — ask for confirmation before launching the workflow:
> "Do you want me to run a full multi-AI research session on [topic]? This will query Claude, Gemini, ChatGPT, and Grok. Should I start?"

**Resume trigger:** If the user says `Resume research` or `Continue research`, see Resume Rules below.

**Cancel trigger:** If the user says "stop", "cancel", or sends an unrelated message while a research session is running, see Cancellation Rules below.

**Setup trigger:** If the user says any of: "setup", "set up", "get started", "let's get started", "how do I start", "how do I begin", "first time setup", "install", "initialize", or anything clearly meaning the same thing — run the **First-Time Setup** flow described at the bottom of this file. Do NOT launch a research session.

Extract the topic from the user's message. The topic is everything after the trigger phrase.

## Local Configuration

This project reads runtime configuration from `config.local.json` at the project root. This file is gitignored. A template lives at `config.example.json`.

**At the start of every research session and every setup run, read `config.local.json`.** If it does not exist, the user has not finished setup — direct them to the First-Time Setup flow.

Config keys:
- `chrome_profile_name` (string, required) — name of the Chrome profile signed into all 4 AI platforms.
- `notion.enabled` (bool) — if false or missing, **skip every Notion step in this file** (do not create rows, do not update statuses, do not append archive toggles). The workflow runs locally only.
- `notion.database_id`, `notion.data_source_id`, `notion.dashboard_page_id` — used only when `notion.enabled` is true.

Throughout this document, references like "the AI Research profile" mean **the profile name from `config.local.json`**, and references to specific Notion IDs mean **the IDs from `config.local.json`**.

## Status Vocabulary

These are the only valid status values used throughout the system. Every reference to a status in the manifest, Notion, resume logic, and reporting must use one of these exact values.

**Platform statuses (per-platform in manifest):**
| Status | Meaning | Set when |
|--------|---------|----------|
| `pending` | Not yet started | Session creation |
| `running` | Currently executing | Platform step begins |
| `success` | Completed successfully | Response extracted and saved |
| `failed` | Hard failure (CAPTCHA, login error, crash, etc.) | Unrecoverable error during execution |
| `timed_out` | Stalled — 5 min of no visible change, retry also stalled | Both attempts showed no activity |
| `skipped` | Deliberately skipped (unavailable at preflight) | User chose to proceed without this platform |
| `cancelled` | User interrupted the workflow | User cancelled mid-run |

**Session statuses (top-level in manifest):**
| Status | Meaning |
|--------|---------|
| `running` | Workflow in progress |
| `synthesizing` | All platforms done, report being generated |
| `completed` | All 4 platforms succeeded, report generated |
| `completed_with_errors` | Report generated but 1+ platforms failed/timed_out/skipped |
| `failed` | No platforms succeeded or report could not be generated |
| `cancelled` | User interrupted the workflow |

## Tab Management

All browser operations require a tab ID. Use **4 parallel tabs** — one per platform.

1. During Step 4 (Preflight), call `tabs_context_mcp` with `createIfEmpty: true` to get or create the MCP tab group.
2. Call `tabs_create_mcp` **4 times** to create one tab per platform. Store each tab ID:
   - `claudeTabId`, `geminiTabId`, `chatgptTabId`, `grokTabId`
3. Use each platform's dedicated tab for ALL operations on that platform: preflight checks, prompt submission, polling, extraction.
4. If a tab becomes unresponsive or is accidentally closed, create a new one with `tabs_create_mcp` and update the corresponding tab ID.
5. Every browser MCP call must explicitly pass the correct platform tab ID as its `tabId` parameter.

**Downloads folder:** Chrome must be configured to download to the `Downloads/` folder inside this project root. If downloads aren't appearing there, check Chrome's download location setting before assuming extraction failed. (The first-time setup flow walks the user through this.)

## Full Workflow

### Step 0: Pre-Run Analysis (Recommendation-Only Mode)

Before creating a new session, check the learning ledger for accumulated data.

1. Read `learning/ledger.json`. Count entries where `low_confidence` is false.
2. If fewer than 5 qualifying entries: show "Learning system: N sessions recorded, need 5+ for analysis." Skip to Step 1.
3. If 5+ qualifying entries:

   a. Compute averages for the last 5 qualifying sessions (overall). If 3+ sessions exist in the current topic's category, also compute category-specific averages.

   b. Show the performance summary:
   ```
   === Research System Performance (last 5 sessions) ===
   Brief Fidelity: X.X | Synthesis Focus: X.X
   Response Relevance: C:X.X G:X.X X:X.X K:X.X (avg X.X)
   Source Relevance: C:X.X G:X.X X:X.X K:X.X (avg X.X)
   [Category-specific averages if available]
   [Active recommendations from improvements.md if any]
   ```

   c. **Identify the weakest primary dimension** (Brief Fidelity, Response Relevance, Synthesis Focus). Source Relevance is tracked but does NOT trigger recommendations on its own — it must be supported by a low Response Relevance score for the same platform.

   d. **If the weakest dimension averages below 3.5, run the diagnosis tree:**

   | If weakest is... | Check upstream | Diagnosis |
   |-----------------|----------------|-----------|
   | Response Relevance (platform X) | Brief Fidelity also < 3.5? | Yes → brief problem, not platform |
   | Response Relevance (platform X) | Any partial extractions for X in recent sessions? | Yes → data loss, not quality. **Not safe to recommend.** |
   | Response Relevance (platform X) | X used fallback mode in recent sessions? | Yes → mode issue, not prompt. **Not safe to recommend.** |
   | Response Relevance (platform X) | Brief Fidelity >= 4.0, no extraction issues? | Platform prompt problem for X |
   | Synthesis Focus | Response Relevance avg >= 4.0? | Synthesis instructions problem |
   | Synthesis Focus | Response Relevance also low? | Upstream problem — fix response relevance first |
   | Brief Fidelity | Only low dimension? | Brief generation problem |
   | Brief Fidelity | Downstream also low? | Brief problem cascading — fix brief first |

   **Tie-break rule:** If multiple platforms/dimensions are close (within 0.3), prioritize the one with lower Response Relevance — that's closest to the core "did it answer the question" metric.

   e. **Confidence check — ALL must be true to log a recommendation:**
      - Weakness appeared in at least 3 of the last 5 qualifying sessions (not a one-off)
      - Diagnosis tree identified a single target stage with no conflicting upstream signals
      - No partial extractions in the sessions being analyzed for this dimension
      - At least 5 qualifying sessions exist (or 3 in the relevant topic category)

   f. **If confidence check passes:** Log a recommendation in `learning/improvements.md`:
   ```
   ## [DATE] Recommendation: [target stage] — [brief description]
   **Weakness:** [dimension] averaged [score] over sessions [list]
   **Diagnosis:** [which stage, why]
   **Suggested change:** [specific text to add/modify]
   **Confidence:** [high/medium] — [which conditions were met]
   **Status:** Pending review
   ```

   **If confidence check fails:** Log an observation instead:
   ```
   ## [DATE] Observation (confidence too low to recommend)
   **Weakness:** [dimension] averaged [score]
   **Why not recommended:** [which condition failed]
   **What to watch:** [what would make this actionable]
   ```

4. **Current mode: recommendation-only.** Step 0 does NOT create adjustments in `adjustments.json`. It only logs recommendations and observations to `improvements.md` and shows the performance summary. Auto-adjustment creation will be enabled in a future update after the scoring system is validated.

5. Check `learning/adjustments.json` — if `active_adjustment` is not null (manually created), increment `sessions_since_applied`. Otherwise, proceed.

### Step 1: Create Session

1. Generate a session ID: `YYYY-MM-DD-HHMM-topic-slug`
   - Use current date and time
   - Slugify the topic: lowercase, replace spaces with hyphens, remove special characters, truncate to 50 chars
   - Example: `2026-04-05-1430-quantum-computing-advances`

2. Create the session directory structure:
   ```
   research/<session-id>/
   ├── manifest.json
   ├── research_brief.md
   ├── preflight.md
   ├── prompts/
   ├── responses/
   └── errors/
   ```

3. Initialize `manifest.json`:
   ```json
   {
     "topic": "",
     "research_title": "",
     "raw_user_request": "<the user's original verbatim input>",
     "session_id": "<session-id>",
     "created_at": "<ISO timestamp>",
     "completed_at": null,
     "status": "running",
     "platforms": {
       "claude": {
         "status": "pending",
         "mode_used": null,
         "started_at": null,
         "ended_at": null,
         "attempts": 0,
         "partial_extraction": false,
         "prompt_shortened": false,
         "prompt_file": "prompts/claude_prompt.md",
         "response_file": "responses/claude_response.md",
         "error_file": null,
         "error_screenshot": null,
         "notes": ""
       },
       "gemini": {
         "status": "pending",
         "mode_used": null,
         "started_at": null,
         "ended_at": null,
         "attempts": 0,
         "partial_extraction": false,
         "prompt_shortened": false,
         "prompt_file": "prompts/gemini_prompt.md",
         "response_file": "responses/gemini_response.md",
         "error_file": null,
         "error_screenshot": null,
         "notes": ""
       },
       "chatgpt": {
         "status": "pending",
         "mode_used": null,
         "started_at": null,
         "ended_at": null,
         "attempts": 0,
         "partial_extraction": false,
         "prompt_shortened": false,
         "prompt_file": "prompts/chatgpt_prompt.md",
         "response_file": "responses/chatgpt_response.md",
         "error_file": null,
         "error_screenshot": null,
         "notes": ""
       },
       "grok": {
         "status": "pending",
         "mode_used": null,
         "started_at": null,
         "ended_at": null,
         "attempts": 0,
         "partial_extraction": false,
         "prompt_shortened": false,
         "prompt_file": "prompts/grok_prompt.md",
         "response_file": "responses/grok_response.md",
         "error_file": null,
         "error_screenshot": null,
         "notes": ""
       }
     },
     "platforms_succeeded": 0,
     "platforms_failed": 0,
     "platforms_timed_out": 0,
     "platforms_skipped": 0,
     "report_file": "report.md"
   }
   ```

4. Tell the user: "Starting research session: `<session-id>`. Refining your topic..."

### Step 2: Refine the Research Topic

The user will provide a casual, conversational description of what they want researched (typically 2-8 sentences). Your job is to distill this into a structured research brief BEFORE it reaches any platform.

**Do NOT pass the user's raw input to the platforms verbatim.**

**Clarify, don't redirect.** Your job is to sharpen the user's question, not change it into a different question. If they asked about cat communication, the brief should be about cat communication — not animal cognition broadly. Stay faithful to what they actually want to know.

1. Read the user's raw input carefully. Identify:
   - The core question or topic they're actually asking about
   - Specific angles or sub-questions they mentioned (explicitly or implicitly)
   - Any scope boundaries (what they care about vs. don't care about)
   - The depth they seem to want (surface overview vs. deep dive)

2. Generate two things:

   **a) Research Title** (short, clean — used for folder names, report headers):
   - Example: "Feline Communication and Human Language Comprehension"

   **b) Research Brief** (the detailed version — inserted into prompts):
   Write a clear, well-structured research brief that includes:
   - **Core question:** One sentence framing the central research question
   - **Key sub-questions:** 3-6 specific questions to investigate
   - **Scope:** What's in bounds and what's out of bounds
   - **Context:** Any relevant framing the user provided

3. Save the research brief to `research/<session-id>/research_brief.md`.
   Update `manifest.json`: set `topic` to the core question and `research_title` to the generated title.

4. Show the user the research brief and title. Say:
   > "Here's how I've framed your research. Starting queries now — I'll update you as each platform completes."

   Do NOT wait for approval — proceed immediately. The brief is shown for transparency, not as a gate. If the user wants to adjust it, they can interrupt or start a new session.

### Step 3: Generate Prompts

1. Read `prompts/system.md` — this is the shared context.
2. Render the system prompt: take the contents of `system.md` and replace `{{TOPIC}}` with the full research brief from Step 2. Call this the **rendered system prompt**.
3. Read each platform template: `prompts/claude.md`, `prompts/gemini.md`, `prompts/chatgpt.md`, `prompts/grok.md`.
4. For each platform template: replace `{{SYSTEM_PROMPT}}` with the rendered system prompt from step 2. The platform templates contain no other placeholders. The result is the final rendered prompt.
5. **Apply adjustment overlay (if any):** Read `learning/adjustments.json`. If `active_adjustment` is not null and its `target_file` matches the current prompt template (and `topic_category` matches if set), append the adjustment's `content` to the rendered prompt. The base template file is never modified — the adjustment is applied only to the rendered output.
6. Save rendered prompts (with any adjustment applied) to the session folder:
   - `research/<session-id>/prompts/claude_prompt.md`
   - `research/<session-id>/prompts/gemini_prompt.md`
   - `research/<session-id>/prompts/chatgpt_prompt.md`
   - `research/<session-id>/prompts/grok_prompt.md`

### Step 4: Preflight Check

Set up the browser automation tab and check each platform's availability.

1. **Verify Chrome profile:** Before creating tabs, check that the connected browser is the profile named in `config.local.json` -> `chrome_profile_name`. If `switch_browser` was not called or the wrong profile is connected, call `switch_browser` and wait for confirmation that the configured profile is connected. Do NOT proceed with the wrong profile — platform logins are profile-specific.

2. **Create automation tabs:** Follow the Tab Management instructions above. Create the tab group and 4 dedicated tabs (one per platform). Store `claudeTabId`, `geminiTabId`, `chatgptTabId`, `grokTabId`.

3. For each platform, navigate its dedicated tab to the platform URL and check login state. Before checking, dismiss any visible cookie consent banners, "welcome back" dialogs, or notification prompts by looking for dismiss/close/decline buttons. If a CAPTCHA or bot-detection challenge appears, mark that platform as failed.

   - **Claude.ai** (`claude.ai`): Look for chat input area, user avatar, or model selector.
   - **Gemini** (`gemini.google.com`): Look for chat input area, user avatar.
   - **ChatGPT** (`chatgpt.com`): Look for message input box, user menu.
   - **Grok** (`grok.com`): Look for chat input, user indicators.

4. Write `preflight.md`:
   ```markdown
   # Preflight Check — <timestamp>

   | Platform  | Status       | Notes |
   |-----------|-------------|-------|
   | Claude.ai | ready/failed | ...   |
   | Gemini    | ready/failed | ...   |
   | ChatGPT   | ready/failed | ...   |
   | Grok      | ready/failed | ...   |
   ```

5. **Decision point:**
   - If all platforms ready: proceed automatically.
   - If 1-2 platforms unavailable: tell the user which ones failed, ask if they want to proceed or fix logins first.
     - If the user says proceed: set the unavailable platforms' status to `"skipped"` in the manifest with notes explaining why. Increment `platforms_skipped`.
     - If the user says fix logins: pause and wait, then re-run preflight for those platforms.
   - If ALL platforms unavailable (Chrome disconnected): stop and tell the user to connect Chrome.

6. **Create the Notion row** — *only if `config.local.json` -> `notion.enabled` is true. Otherwise skip this step entirely.* Use the `database_id` and `data_source_id` from config.
   - Topic: research_title
   - Status: "Running"
   - Claude / Gemini / ChatGPT / Grok: "Pending" (or "Skipped" if skipped in preflight)
   - Platforms: "0/4"
   - Started: current datetime
   - Session ID: the session folder name
   - **Store the Notion page ID** returned by the create call in the manifest under a `"notion_page_id"` field. This avoids needing to search for the row during later updates.

**Important:** Preflight results are point-in-time. Execution steps must still handle login failures gracefully since session state can change between preflight and execution.

### Step 5: Execute Platform Queries

Run all 4 platforms **in parallel** using their dedicated tabs. Submit prompts to all 4 platforms, then poll each tab for completion, extracting responses as each platform finishes.

**All 4 platforms are browser-automated.** Each gets a fresh conversation with just the research prompt. No platform sees another's output. Claude Code's role is orchestration and synthesis only.

**Execution rules that apply to ALL platforms:**
- Before each platform step, check if the platform's status is `"skipped"`. If so, skip it entirely.
- **Every execution step must re-navigate to its target URL.** Never assume the browser is still on the page left by preflight.
- Always start a fresh conversation. Never reuse an existing chat.
- Extract only the final assistant response, not the full page (no navigation elements, sidebars, or page chrome).
- **Every response file must include source URLs.** After extraction, verify that sources are present. If they're missing, extract them from the platform's DOM before moving on.
- **Error files** use the naming convention: `errors/<platform>_error.md` (e.g., `errors/claude_error.md`, `errors/chatgpt_error.md`). Update `error_file` in manifest.
- **Polling (parallel):** After submitting prompts to all 4 platforms, cycle through each tab checking for completion. Use `get_page_text` or a quick screenshot on each tab. Check each tab roughly every 60-90 seconds. A platform is "done" when the response is fully rendered, any stop/generating button is gone, and no new text is appearing. Start extraction on each platform as it completes — don't wait for all 4.
- **Prompt length:** After entering the prompt in the input field, verify the full text was accepted. If the input appears truncated, shorten the prompt by removing sub-questions from the research brief (last sub-question first), preserving the core question and scope. The full original prompt is already saved as `prompts/<platform>_prompt.md`. Save the shortened version that was actually submitted as `prompts/<platform>_submitted_prompt.md` (this file is only created when shortening was needed). Set `prompt_shortened: true` in manifest and note which sub-questions were removed.

**Platform-specific input methods (CRITICAL — using the wrong method corrupts prompts):**

Each platform's input field requires a different method. Using the wrong one causes silent failures (spaces stripped, text truncated, auto-submit on newline).

| Platform | Input Method | Why Others Fail |
|----------|-------------|-----------------|
| **Claude.ai** | JS `innerHTML` with `<p>` tags on contenteditable div | `form_input` fails (DIV not supported) |
| **Gemini** | Click input ref → `type` action | JS innerHTML blocked (TrustedHTML), form_input fails (DIV), execCommand fails silently |
| **ChatGPT** | JS `execCommand('selectAll'); execCommand('delete');` then `execCommand('insertText', false, text)` on `#prompt-textarea` | `type` action auto-submits on newline. Use single-paragraph format (no newlines) |
| **Grok** | React native setter hack on textarea (see below) | `type` action strips ALL spaces; `form_input` sets value but React doesn't see it; `execCommand` only inserts 1 char |

**Grok native setter method:**
```javascript
const el = document.querySelector('textarea');
const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
nativeSetter.call(el, promptText);
el.dispatchEvent(new Event('input', { bubbles: true }));
el.dispatchEvent(new Event('change', { bubbles: true }));
```

**Mandatory prompt verification (ALL platforms, no exceptions):**
1. **Before sending:** After entering the prompt, verify the full text is present via JavaScript key-phrase checks. Check at least 3 phrases from different parts of the prompt (beginning, middle, end). Check total character length. Do NOT click send until verification passes.
2. **After sending:** Screenshot or read the sent message to confirm the full prompt appears in the conversation. If truncated, stop the response and resubmit with the complete prompt in a fresh conversation.

**Parallel execution order:**
1. Navigate all 4 tabs to their platform URLs and start fresh conversations.
2. Activate each platform's research/expert mode and submit prompts to all 4.
3. Poll all tabs in rotation, extracting each response as it completes.
4. Update manifest and Notion after each platform finishes.

**Expected research durations (for stall calibration):**
| Platform | Typical Duration | Notes |
|----------|-----------------|-------|
| Grok Expert | 5-15 minutes | Fastest; single thinking phase |
| Claude Research | 8-10 minutes | Shows source count increasing |
| ChatGPT Deep Research | 20-30 minutes | Shows research progress steps |
| Gemini Deep Research | 30-60+ minutes | Longest; shows "Sources found" count |

Do not flag a platform as stalled just because it's taking longer than expected. Only flag based on the 5-minute no-change rule below.

**Stall detection and retry (same for all platforms):**
- If nothing on the page has changed for **5 consecutive minutes**, the platform is stalled.
- On first stall: set manifest notes to describe the stall, wait 30 seconds, refresh the page, start a fresh conversation, resubmit the same prompt (attempt #2).
- On second stall (retry also showed no activity for 5 min): mark as `"timed_out"`, save error details to `errors/<platform>_error.md`, take a screenshot (save path to `error_screenshot`), increment `platforms_timed_out`, continue to next platform.
- There is NO absolute timeout. If a platform is still actively generating (text appearing, progress updating, source count increasing), let it run as long as it needs — even over an hour.

**Hard failure handling (same for all platforms):**
- CAPTCHA, login error, crash, error page, or unexpected state: mark as `"failed"`, save error to `errors/<platform>_error.md`, take screenshot, increment `platforms_failed`, continue to next platform.

#### 5a: Claude.ai (Research Mode)

1. Update manifest: claude status -> `"running"`, set started_at, increment attempts.
2. In `claudeTabId`, navigate to `claude.ai`.
3. Start a fresh conversation.
4. **Activate Research mode:**
   - Click the `+` button in the input area
   - Click "Research" from the menu
   - A research icon should appear next to the `+` button confirming activation
   - Set mode_used -> `"research"`
   - If "Research" option is not found in the menu: use standard Claude, set mode_used -> `"standard"`, add note.
   - **Connector popup:** If Claude shows an "Enable connectors for Claude to use in research" popup, click "Disable all tools". The popup may appear a SECOND time with Notion toggle now disabled — click "Confirm" on the second appearance. Do NOT enable Notion or any other connector. Do NOT click "Cancel Research" — this cancels the prompt entirely.
5. Enter the rendered prompt from `prompts/claude_prompt.md` and submit.
6. **Claude may ask clarifying questions** before starting research. If it does, answer them based on the research brief (accept its proposed research plan or confirm its direction). Then wait for the full research to run.
7. Wait for response (see polling and stall rules above).
8. **Extract response — Claude.ai requires a two-step extraction:**
   a. First verify the artifact panel is open on the right side of the page. If it's not visible, click on the artifact card/document block in the chat conversation to reopen it.
   b. Use `read_page` with `filter: interactive` to find the artifact panel buttons. Locate the "Copy options" button by its ref (do NOT guess coordinates for small UI elements — always click by ref).
   b. Click "Copy options" ref, then click "Download as Markdown" from the dropdown.
   c. Read the downloaded `.md` file from `Downloads/` folder.
   d. The markdown has the full report text BUT source citation badges are plain text without URLs. Extract source URLs from the artifact DOM: call `read_page` on the artifact panel content area ref, then collect all `<a href>` elements — these are the inline citation links.
   e. Deduplicate the URLs and append a `## Sources` section to the response file.
   f. Save the final file to `responses/claude_response.md`.
   g. Delete the downloaded markdown from `Downloads/` (intermediate artifact).
9. Update manifest: status -> `"success"`, set ended_at.
10. Update Notion: Claude -> "Done", Platforms count updated.
11. Tell the user: "Claude: done."

#### 5b: Gemini Deep Research

1. Update manifest: gemini status -> `"running"`, set started_at, increment attempts.
2. In `geminiTabId`, navigate to `gemini.google.com`.
3. Start a fresh conversation.
4. **Activate Deep Research mode:**
   - Ensure "Thinking" is selected in the mode dropdown (right side of input)
   - Click the "Tools" button (to the right of the `+` button)
   - Click "Deep research" from the tools menu
   - A "Deep research" badge should appear in the input area confirming activation
   - Set mode_used -> `"deep_research"`
   - If "Deep research" is not in the tools menu: use standard Gemini, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt using the `type` action (click input ref first, then type). Do NOT use JavaScript injection — Gemini's TrustedHTML policy blocks `innerHTML` and `execCommand` fails silently. Verify full text is present before submitting.
6. Click the send button. **If the page refreshes instead of processing:** immediately check network requests via `read_network_requests` for a 503 status on the `StreamGenerate` endpoint. If 503 is present, this is server-side rate limiting — do NOT waste time trying different click methods (ref clicks, coordinate clicks, JS clicks, keyboard shortcuts all fail equally). Mark Gemini as `"failed"` and move on. The 503 can be triggered by rapid automated page loads and JS manipulation.
7. **Gemini will present a research plan.** Click "Start" or the equivalent approval button to begin the deep research. Wait for research to complete — this can run 30+ minutes or over an hour. This is normal if progress indicators are updating.
7. **Extract response — Gemini requires JavaScript extraction:**
   a. Do NOT use "Export to Docs" — it shows "Creating document..." indefinitely and never completes.
   b. Do NOT use `get_page_text` — Gemini Deep Research responses exceed the 50K character tool limit.
   c. Use JavaScript to get the response: `document.querySelector('.response-container-content').innerText` (typically ~49K chars).
   d. The JS tool output truncates at ~3K chars, so you cannot read it in one call. Split into chunks first:
      ```javascript
      const text = document.querySelector('.response-container-content').innerText;
      const chunkSize = 3000;
      window.__gem_chunks = [];
      for (let i = 0; i < text.length; i += chunkSize) {
        window.__gem_chunks.push(text.substring(i, i + chunkSize));
      }
      `Created ${window.__gem_chunks.length} chunks`;
      ```
      Then concatenate all chunks into a Blob and trigger download:
      ```javascript
      const fullText = window.__gem_chunks.join('');
      const blob = new Blob([fullText], {type: 'text/plain'});
      const a = document.createElement('a');
      a.href = URL.createObjectURL(blob);
      a.download = 'gemini_response.txt';
      a.click();
      ```
   e. Read the downloaded file. It will contain three sections in order:
      - **Report content** → KEEP
      - **"Sources used in the report"** + **"Sources read but not used"** (URLs with page titles, interleaved with "Opens in a new window") → KEEP (strip "Opens in a new window" lines)
      - **"Thoughts"** section (Gemini's internal thinking: "I am synthesizing...", "I am resolving...") → DISCARD everything from "Thoughts" onward
   f. Clean and save to `responses/gemini_response.md`. Delete the downloaded `.txt` from `Downloads/`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: Gemini -> "Done", Platforms count updated.
10. Tell the user: "Gemini: done."

#### 5c: ChatGPT

1. Update manifest: chatgpt status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `chatgptTabId`, navigate to `chatgpt.com`.
3. Start a fresh conversation.
4. **Activate Deep Research mode:**
   - Click the `+` button in the input area
   - Click "Deep research" from the menu
   - A "Deep research" badge should appear confirming activation
   - The input placeholder should change to "Get a detailed report"
   - Set mode_used -> `"deep_research"`
   - If "Deep research" is not in the menu: use standard ChatGPT, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt from `prompts/chatgpt_prompt.md` and submit.
6. **ChatGPT may present a research plan to approve.** Click the approval/start button to begin. Wait for response (see polling and stall rules above).
7. **Extract response — ChatGPT requires .docx download via fullscreen view:**
   a. Do NOT try to extract from the DOM — ChatGPT Deep Research renders in a **cross-origin iframe** (`web-sandbox.oaiusercontent.com`). JavaScript cannot access it. `get_page_text` only captures the main page metadata, not the report content.
   b. Do NOT download as markdown — ChatGPT's markdown export strips all source citation URLs.
   c. **Open fullscreen view:** Below the document panel, find the response action buttons (copy, thumbs up, thumbs down, share, switch model, more). One of these icons (or clicking the document panel itself) expands the document to fullscreen.
   d. **Download from fullscreen:** In the fullscreen view, click the **download icon** (downward arrow in a circle) at the **top-right corner** of the page. A dropdown will appear with options: "Copy contents", "Export to Markdown", "Export to Word", "Export to PDF". Click **"Export to Word"** — this preserves citation numbers AND their full URLs.
   e. Read the downloaded `.docx` file from `Downloads/`. Parse the XML inside using **Perl** (portable across Windows Git Bash, macOS, and Linux — do NOT use `sed 's/.../\n/g'` because BSD sed on macOS treats `\n` as a literal `n`):
      ```bash
      unzip -p file.docx word/document.xml | perl -pe 's/<w:p[^>]*>/\n/g; s/<[^>]*>//g' | grep -v '^[[:space:]]*$' > responses/chatgpt_response.md
      ```
   f. Verify the extracted file contains citation numbers (e.g., `[1]`, `[41]`) in the text AND full URLs at the end. If URLs are missing, re-download as .docx.
   g. Delete the `.docx` and any intermediate files from `Downloads/` and `responses/`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: ChatGPT -> "Done", Platforms count updated.
10. Tell the user: "ChatGPT: done."

#### 5d: Grok

1. Update manifest: grok status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `grokTabId`, navigate to `grok.com`.
3. Start a fresh conversation.
4. **Set Expert mode:**
   - Click the mode selector dropdown (shows current mode like "Expert" or "Fast")
   - Select "Expert" (Thinks hard - Grok 4.20) if not already selected
   - Set mode_used -> `"expert"`
   - Grok does not have a separate research mode — it will do research when prompted to do so.
5. Enter the rendered prompt from `prompts/grok_prompt.md` and submit.
6. Wait for response (see polling and stall rules above).
7. **Extract response — Grok is the simplest extraction:**
   a. Use `get_page_text` — this works cleanly for Grok and captures the full response within the character limit.
   b. Extract additional source URLs via JavaScript from the Sources panel/sidebar if present.
   c. Save the response text with all sources appended to `responses/grok_response.md`.
   d. Note: Grok reports "N sources" in its header (e.g., "68 sources") but typically only lists 8-10 explicit URLs. The remaining sources were used during thinking but not surfaced. This is a Grok limitation, not an extraction failure.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: Grok -> "Done", Platforms count updated.
10. Tell the user: "Grok: done."

**After each platform completes, fails, or times out**, tell the user:
> "[Platform]: [done/failed/timed out] ([brief reason if applicable])."

And update Notion for that platform accordingly.

When all platforms are done (or failed/timed out), tell the user:
> "All platforms complete. Synthesizing report..."

### Step 6: Synthesize Report

1. Update manifest: status -> `"synthesizing"`.
2. Update Notion: Status -> "Synthesizing".
3. Read ALL response files from `research/<session-id>/responses/` (only those with status `"success"` in manifest).
4. Read the report template from `templates/report.md` and use it as a **structural guide** — follow its section headings and overall shape, but write naturally. Do not do mechanical find-and-replace on placeholder strings.
5. Write the report following this structure:

   **Title and metadata:** Research title, date, session ID, which platforms responded, which were unavailable and why.

   **Executive Summary:** 2-3 paragraphs synthesizing the most important findings ACROSS all sources. Identify the overarching narrative — do not just summarize each AI separately.

   **Key Findings:** Organize by THEME, not by platform. Use 3-6 themes depending on topic complexity. Weave insights from multiple sources together under each theme.

   **Consensus & Disagreement:**
   - Agreement: where 2+ platforms reached similar conclusions (higher confidence)
   - Disagreement: where platforms contradicted each other (flag for verification)
   - Verification needed: claims from only one source that could not be confirmed

   **Unique Insights:** Things only one platform surfaced. Attribute clearly: "Grok noted that..." or "Gemini found a source indicating..."

   **Sources & Citations:** Consolidate all URLs, paper references, and named sources. Label as **"Sources cited by platforms (not independently verified)."**

   **Further Questions:** Follow-up research topics raised by this investigation.

   **Confidence Assessment:**
   - How many platforms responded out of 4
   - Which modes were used (Research/Deep Research/standard/free)
   - Whether any prompts were shortened
   - Whether any extractions were partial
   - Source quality and topic maturity assessment

   **Appendix:** Table showing each platform's final status and a note that full responses are in the local session folder.

6. Write the completed report to `research/<session-id>/report.md`.

### Step 7: Finalize

1. Update manifest.json:
   - Count `platforms_succeeded`, `platforms_failed`, `platforms_timed_out`, `platforms_skipped`
   - Set overall status:
     - `"completed"` — all 4 platforms succeeded
     - `"completed_with_errors"` — report was generated but 1+ platforms failed/timed_out/skipped
     - `"failed"` — no platforms succeeded or report could not be generated
   - Set `completed_at` timestamp

2. Update Notion row:
   - Status -> "Completed" or "Completed with Errors" or "Failed"
   - Completed: current datetime
   - Platforms -> final count (e.g., "3/4")

3. **Clean up intermediate files:** Delete any intermediate extraction artifacts from `Downloads/` and `responses/` (e.g., `.docx`, `.txt`, `_raw.txt`). Only the final `.md` response files should remain in `responses/`.

4. Add archive toggle to the Notion dashboard page — *only if `config.local.json` -> `notion.enabled` is true. Otherwise skip.* Use the `dashboard_page_id` from config. Place under the "Research Archive" section. Use `<details>/<summary>` HTML tags for Notion toggles. Content inside must be **tab-indented**. `<summary>` must be all on ONE line.

   ```
   <details>
   <summary>[Research Title] (YYYY-MM-DD)</summary>

   	<details>
   	<summary>Executive Summary</summary>

   		[paste executive summary text]

   	</details>

   	<details>
   	<summary>Confidence Assessment</summary>

   		[paste confidence section text]

   	</details>

   	<details>
   	<summary>Platform Results</summary>

   		Claude: [status] | Gemini: [status] | ChatGPT: [status] | Grok: [status]

   	</details>

   	<details>
   	<summary>Session Details</summary>

   		Session ID: [id]
   		Full responses and report stored locally in research/[session-id]/

   	</details>

   </details>
   ```

5. **Do NOT tell the user research is complete yet.** Proceed immediately to Step 7.5 (self-evaluation). The user is only notified of completion AFTER evaluation finishes.

### Step 7.5: Self-Evaluation (Intent Tracing)

**This step is MANDATORY and runs automatically.** Do NOT ask the user whether to run it. Do NOT skip it. Do NOT report session completion until it finishes.

After the report is complete, perform an automated self-evaluation. This traces the user's original intent through the entire pipeline and scores how well it was preserved at each stage.

**Inputs:** Read `raw_user_request` from manifest, `research_brief.md`, all response files (only for platforms with status `"success"`), and `report.md`.

#### Scoring Procedure

**1. Brief Fidelity (always scored):**
   - List every distinct thing the user asked about in `raw_user_request` (call these "user intents"). Count = U.
   - List every sub-question in `research_brief.md`. Count = S.
   - Classify each sub-question:
     - **traced**: maps to a specific phrase the user said (quote both)
     - **inferred**: necessary to answer the user's explicit questions — not just related or helpful, but required. Explain why it's necessary.
     - **system-added**: not traceable and not necessary to answer what the user asked
   - Classify each user intent: **represented** (at least one sub-question covers it) or **missing**.
   - Score: 5 = all traced/inferred + zero missing | 4 = all represented + at most 1 system-added | 3 = >=75% represented + at most 2 system-added | 2 = 1+ missing OR 3+ system-added | 1 = core question reframed OR >50% system-added

**2. Response Relevance (per platform with status "success"):**
   - List all sub-questions from the research brief. Count = Q.
   - For each sub-question, check the platform's response: **substantive** (dedicated section or multiple paragraphs with specific analysis), **passing** (1-2 sentences without depth), or **absent** (not addressed).
   - Count substantive = S, passing = P, absent = A.
   - Score: 5 = S=Q | 4 = S>=75% AND A=0 | 3 = S>=50% AND A<=1 | 2 = S>=25% OR A>=50% | 1 = S<25%
   - Platforms with status failed/timed_out/skipped: score = N/A with reason.

**3. Source Relevance (per platform with status "success"):**
   - Collect all citable sources (URLs + named references). Count = T.
   - If T < 3: score = N/A ("insufficient sources").
   - If T is 3-9: evaluate ALL. If T >= 10: evaluate first 5 + last 5.
   - Classify each: **specific** (directly about the research topic's niche), **generic** (broad overview), or **ambiguous** (can't tell from title/URL).
   - Score based on specific/(evaluated - ambiguous) ratio: 5 = >=80% | 4 = >=60% | 3 = >=40% | 2 = >=20% | 1 = <20%
   - If ambiguous > 50%: cap at 3, note "confidence limited."
   - Source Relevance is a weaker diagnostic signal. It is tracked but does NOT trigger recommendations on its own unless supported by a low Response Relevance score for the same platform.

**4. Synthesis Focus (always scored if report exists):**
   - List all Key Findings subsections in `report.md`. Count = R.
   - Classify each: **on-topic** (directly addresses sub-questions — name which ones), **tangential** (related but not asked for), or **off-topic**.
   - Check executive summary: on-target? (yes/no)
   - Score: 5 = O=R + exec on-target | 4 = O>=80% + Tg<=1 + exec on-target | 3 = O>=60% + Off=0 | 2 = O>=40% OR Off>=1 | 1 = O<40%
   - If only 1 platform succeeded: score normally but add `"limited_evidence": true`.
   - If no report exists: N/A.

**5. Prompt Compliance (secondary, per platform with status "success"):**
   - List deliverables from the platform's prompt template. Count = D.
   - Check each: **present** or **absent**.
   - Score: 5 = all present | 4 = >=75% | 3 = >=50% | 2 = >=25% | 1 = <25%
   - This dimension is secondary — it is tracked but does NOT trigger recommendations on its own. Intent alignment (dimensions 1-4) always takes priority.

#### After Scoring

1. Compute prompt hashes: `cksum prompts/system.md | cut -d' ' -f1` (truncated to 8 chars). Same for each platform template.
2. Auto-classify topic category from manifest topic: tech, design, science, health, business, policy, culture, other.
3. Compute execution metrics from manifest (duration, platform counts, retry count, context exhaustion).
4. Compute per-platform metrics from response files (response_length_chars, source_count, duration_minutes).
5. Write `evaluation.json` to the session folder following the schema in `templates/evaluation_schema.json`. Every score must include its required evidence (the mappings, checklists, and classifications described above).
6. Determine `low_confidence`: true if fewer than 2 platforms succeeded.
7. Append a compressed entry to `learning/ledger.json` (scores, execution metrics, platform summary, prompt hashes, topic category, low_confidence flag).
8. Update the Notion row with: Brief Fidelity, Synthesis Focus, Duration, Eval Summary (compact per-platform string).
9. Add `"evaluation_file": "evaluation.json"` to manifest.json.
10. **NOW tell the user the session is complete:**
    - "Research complete! Report saved to `research/<session-id>/report.md`"
    - Brief summary of which platforms succeeded/failed/timed out
    - Note if any platform used a fallback mode (standard instead of Research/Deep Research)
    - Show the evaluation scores (Brief Fidelity, Synthesis Focus, per-platform Response Relevance)
    - Offer to show the executive summary

#### Review Research Performance

**Trigger:** When the user says `Review research performance`:

1. Read `learning/ledger.json`.
2. If fewer than 3 entries: "Not enough data yet (N sessions). Need at least 3 for meaningful analysis."
3. If 3+ entries: generate a full analysis:
   - Score trends over all sessions (text table)
   - Per-platform performance comparison (average scores, rankings)
   - Category-specific breakdowns (where 3+ sessions exist)
   - Prompt hash change detection (which templates changed between sessions)
   - Weakest dimensions with diagnosis
   - Recommendations from `improvements.md`
   - Adjustment history (if any exist in `adjustments.json`)

## Cancellation Rules

If the user interrupts a running research session (says "stop", "cancel", or sends an unrelated message):

1. Stop the current platform operation immediately.
2. Update the manifest:
   - Set any platform currently `"running"` to `"cancelled"`.
   - Leave `"success"` platforms unchanged.
   - Leave `"pending"` platforms as `"pending"`.
   - Set session status to `"cancelled"`.
3. Update Notion: set the running platform to "Cancelled", set session Status to "Cancelled".
4. Tell the user what was saved and what was not:
   > "Research session cancelled. [N] platforms completed before cancellation. You can resume later with 'Resume research'."
5. Do NOT delete any files. Partial sessions can be resumed.

## Resume Rules

When the user says "Resume research" or "Continue research":

1. Scan `research/` for all session folders. Read each `manifest.json`.
2. Find sessions with status `"running"`, `"failed"`, `"completed_with_errors"`, or `"cancelled"`.
3. **If no resumable sessions exist:** tell the user all sessions are complete and offer to start a new one.
4. **If exactly one resumable session exists:** resume it automatically.
5. **If multiple resumable sessions exist:** list them and ask the user which one to resume:
   > "I found 2 unfinished research sessions:
   > 1. `2026-04-05-1430-quantum-computing` — Gemini failed, others done
   > 2. `2026-04-05-1600-climate-policy` — cancelled after Claude completed
   > Which one should I resume?"

6. To resume a session:
   - Set session status back to `"running"`.
   - Identify which platforms have status `"pending"`, `"failed"`, `"timed_out"`, or `"cancelled"`.
   - Platforms with status `"skipped"` are NOT retried on resume (the user already chose to skip them).
   - Platforms with status `"success"` are NOT re-run.
   - Run preflight for the platforms that need retrying.
   - Execute only those platforms.
   - After all retries complete, re-synthesize the report with all available responses.
   - Update manifest and Notion.
   - To update the existing Notion row (only if `notion.enabled` is true): use the Notion search tool with the session ID as query and the `data_source_id` from `config.local.json` as the `data_source_url`. If no row is found, create a new one.

## Stall Detection Reference

All browser platforms use the same rule: **5 minutes of no visible change = stalled.**

| What counts as "still active" | What counts as "stalled" |
|-------------------------------|--------------------------|
| New text appearing | Page unchanged for 5 min |
| Loading spinner or progress bar moving | Spinner frozen for 5 min |
| Source count increasing (Gemini DR) | No new sources for 5 min |
| Stop/cancel button still visible | Button gone but no final response |

**On first stall:** Retry once in a fresh conversation.
**On second stall:** Mark as `"timed_out"` and move on. Max 2 attempts per platform.

## Error Handling Rules

1. **One platform fails/times out:** Skip it, note in manifest and errors/ folder, continue with the rest. The report states which platforms contributed.

2. **Multiple platforms fail:** Still produce a report with whatever responded, but add a prominent "Limited Confidence" warning.

3. **Chrome disconnected mid-run:** Save whatever has been collected. Update manifest. Tell the user. They can reconnect and use "Resume research."

4. **Unexpected page state:** CAPTCHA, maintenance page, error popup — take a screenshot, save to errors/, mark as `"failed"`, continue.

5. **Partial extraction:** If response text is incomplete, save what you can, set `partial_extraction: true`, note it. A partial response is better than none.

## Notion Integration

**Notion is optional.** If `config.local.json` -> `notion.enabled` is false (or the file is missing), skip every step in this section and every Notion update mentioned elsewhere in this file. The local files are always the source of truth.

When Notion is enabled, the dashboard provides live status tracking and a research archive.

**Database:** the user's "AI Research Sessions" database
- **Database ID:** read from `config.local.json` -> `notion.database_id`
- **Data source ID:** read from `config.local.json` -> `notion.data_source_id`
- **Dashboard page ID:** read from `config.local.json` -> `notion.dashboard_page_id`

**Verified Notion property names** (exact match required):
- `Topic` (title), `Status` (select), `Claude` (select), `Gemini` (select), `ChatGPT` (select), `Grok` (select), `Platforms` (text), `Tags` (multi-select), `Started` (date), `Completed` (date), `Session ID` (text)

**Notion display values:**
- Platform columns: Pending, Running, Done, Failed, Timed Out, Skipped, Cancelled
- Status column: Running, Synthesizing, Completed, Completed with Errors, Failed, Cancelled

**When to update Notion:**
| Workflow moment | Notion update |
|----------------|---------------|
| End of Step 4 (after preflight) | Create row: Topic, Status=Running, all platforms=Pending (or Skipped), Platforms="0/4", Started, Session ID |
| Step 5 platform starts | Platform column -> "Running" |
| Step 5 platform succeeds | Platform column -> "Done", Platforms count updated |
| Step 5 platform fails | Platform column -> "Failed" |
| Step 5 platform times out | Platform column -> "Timed Out" |
| Step 6 starts (synthesis) | Status -> "Synthesizing" |
| Step 7 (finalize) | Status -> final value, Completed date, Platforms -> final count |
| Step 7 (archive) | Add toggle block to dashboard page with inline summary |
| Cancellation | Running platform -> "Cancelled", Status -> "Cancelled" |

**Notion update failures:** If a Notion update fails (API error, rate limit), do NOT stop the research workflow. Log the failure in manifest notes and continue. Local files are the source of truth. Retry failed Notion updates during Step 7.

## Important Notes

- **All platforms are independent.** Each gets a fresh conversation with just the research prompt. No platform sees another's output.
- **Always re-navigate.** Every execution step navigates fresh to its target URL. Never assume the browser is on any particular page.
- **Always start fresh conversations.** Never reuse an existing chat on any platform.
- **Do not rush responses.** Wait for each platform to fully complete. Partial responses are worse than waiting.
- **Keep the user informed.** Status updates after every platform completes or fails.
- **The report is the product.** Spend real effort on synthesis. Raw responses are reference material.
- **Sources are unverified.** AI platforms hallucinate URLs. Always label source lists as unverified.
- **Every response must include sources.** After saving each response file, verify it contains a sources/citations section with actual URLs. If sources are missing, extract them from the platform's DOM before proceeding.
- **Session folders are self-contained.** Old sessions can be deleted without affecting anything.
- **Record what actually happened.** The manifest should accurately reflect reality — modes used, extractions, attempts, shortcuts taken.
- **Never use clipboard for extraction.** `navigator.clipboard.readText()` fails when the tab is not focused (common with multi-tab setups). Always use downloads or direct DOM extraction instead.
- **Never delegate browser automation to background agents.** Background agents (via the Agent tool) may not receive permissions for browser MCP tools (javascript_tool, computer, etc.). Perform all browser operations inline in the main conversation.
- **Click small UI elements by ref, not coordinates.** Use `read_page` to find the exact element reference first, then click by ref. Coordinate-based clicking is imprecise for small buttons like dropdown arrows.
- **Manage context aggressively.** Browser automation consumes large amounts of context. Once a platform response is extracted and saved, do not re-read the full response unless needed for synthesis. Minimize screenshot frequency — only screenshot when state is unknown. Use `read_page` with specific ref_ids instead of full page reads.
- **Mid-session context recovery.** If the context window fills up and the session continues in a new conversation, reconstruct state by: (1) reading `manifest.json` for platform statuses and session metadata, (2) checking which response files exist in `responses/`, (3) resuming from the first platform that isn't `"success"`. The manifest is the source of truth for what's been completed.
- **User interaction during research.** The user may interact with the browser while research is running (browsing other tabs, etc.). This is generally safe. However, switching to or closing one of the 4 platform tabs can break polling or extraction. If a platform tab becomes unresponsive or navigates away, create a new tab and restart that platform.

## First-Time Setup

Triggered when the user says "setup", "set up", "get started", "let's get started", "how do I start", "how do I begin", "first time setup", "install", "initialize", or anything clearly meaning the same thing.

**Use the `AskUserQuestion` tool to run a short questionnaire — NOT a text list.** This pops up a native multiple-choice / yes-no UI in Claude Code so the user can click answers instead of typing them.

### Step A: Run the questionnaire

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

### Step B: Act on the answers

Run ONLY the tasks the user selected. Each task block below is independent. Keep prose minimal — be concise.

**Store the OS answer.** Use it to format every example path you show the user. The actual project root is your current working directory — use it when possible. Only fall back to the generic examples below if the cwd isn't obvious.

Path examples by OS (replace `<project>` with the actual project path when you know it):
- **Windows:** `C:\path\to\ai-research-agent\Downloads\` — backslashes for display to the user, forward slashes inside bash commands
- **macOS:** `/Users/<you>/path/to/ai-research-agent/Downloads/`
- **Linux:** `/home/<you>/path/to/ai-research-agent/Downloads/`

#### Task: Create folders + config

1. Check which of these exist at the project root and create only what's missing: `prompts/`, `templates/`, `learning/`, `research/`, `Downloads/`.
2. If `config.local.json` is missing, copy `config.example.json` to `config.local.json`.
3. If `learning/ledger.json` is missing, create it: `{"ledger_version": "1.0", "sessions": [], "promotions": []}`.
4. If `learning/adjustments.json` is missing, create it: `{"adjustments_version": "1.0", "mode": "recommendation_only", "active_adjustment": null, "history": [], "pending_recommendations": []}`.
5. Tell the user one line: "Folders ready. Open `config.local.json` and set `chrome_profile_name`."

#### Task: Walk through Chrome setup

Show these steps numbered, nothing else:

1. Install the **Claude in Chrome** extension from the Chrome Web Store.
2. In Claude Code, run `/mcp` to confirm the Claude in Chrome MCP server is connected. If it isn't, follow the prompts to add it — the agent cannot drive the browser without it.
3. Create a dedicated Chrome profile for research (recommended name: `AI Research`) via Chrome menu -> profile picker -> Add.
4. In that profile's Chrome settings -> Downloads -> Location, set the download folder to the project's `Downloads/` directory. Show the OS-specific path.
5. **Heads up:** Chrome's download location is per-profile, so every download in this profile will land in the project folder. That's why a dedicated profile is strongly recommended.
6. Put the profile name into `config.local.json` -> `chrome_profile_name`.

#### Task: Account checklist

Show this list and nothing else:

1. **Claude.ai** — Pro or higher (Research mode is paywalled).
2. **Gemini** — Google account with Gemini Advanced (Deep Research requires Advanced).
3. **ChatGPT** — Plus or higher (Deep Research is paywalled).
4. **Grok** — grok.com account with Expert mode access.

Then: "Sign into all four in your research Chrome profile. Open each site once to verify no login prompt appears."

#### Task: Notion integration

1. Install or connect the **Notion MCP** in Claude Code (run `/mcp` and add it). Grant it access to a workspace you control.
2. In that workspace, create a new database called **"AI Research Sessions"** with these exact properties: `Topic` (title), `Status` (select), `Claude` (select), `Gemini` (select), `ChatGPT` (select), `Grok` (select), `Platforms` (text), `Tags` (multi-select), `Started` (date), `Completed` (date), `Session ID` (text).
3. Status select options: `Running`, `Synthesizing`, `Completed`, `Completed with Errors`, `Failed`, `Cancelled`. Each platform select needs: `Pending`, `Running`, `Done`, `Failed`, `Timed Out`, `Skipped`, `Cancelled`.
4. Create a separate page called something like **"AI Research Dashboard"** — session archive toggles will be appended here after each run.
5. Collect three IDs:
   - **Database ID** — the 32-char hex string in the database URL (before `?`).
   - **Data source ID** — after connecting the Notion MCP, ask Claude to call `notion-fetch` on the database URL and read the returned `data_source_id`.
   - **Dashboard page ID** — same format, from the dashboard page URL.
6. Edit `config.local.json`: set `notion.enabled` to `true` and paste the three IDs into `database_id`, `data_source_id`, `dashboard_page_id`.

### Step C: Wrap up

Finish with one short line: "Setup tasks done. To start a research session, type `Research: <your topic>`."

Do NOT auto-launch a research session after setup. Wait for the user to trigger one.
