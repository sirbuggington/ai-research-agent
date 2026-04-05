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

Extract the topic from the user's message. The topic is everything after the trigger phrase.

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

All browser operations require a tab ID. Use a single dedicated automation tab.

1. During Step 4 (Preflight), call `tabs_context_mcp` with `createIfEmpty: true` to get or create the MCP tab group.
2. Call `tabs_create_mcp` to create one dedicated automation tab. Store its `tabId` — this is `automationTabId`.
3. Use `automationTabId` for ALL browser operations throughout the session: preflight checks, platform queries, everything.
4. Do NOT create a new tab per platform. Reuse the same tab — each navigation replaces the previous page.
5. If a tab becomes unresponsive or is accidentally closed, create a new one with `tabs_create_mcp` and update `automationTabId`.
6. Every browser MCP call must explicitly pass `automationTabId` as its `tabId` parameter.

## Full Workflow

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
5. Save rendered prompts to the session folder:
   - `research/<session-id>/prompts/claude_prompt.md`
   - `research/<session-id>/prompts/gemini_prompt.md`
   - `research/<session-id>/prompts/chatgpt_prompt.md`
   - `research/<session-id>/prompts/grok_prompt.md`

### Step 4: Preflight Check

Set up the browser automation tab and check each platform's availability.

1. **Create automation tab:** Follow the Tab Management instructions above. Create the tab group and the dedicated automation tab. Store `automationTabId`.

2. For each platform, navigate `automationTabId` to the platform URL and check login state. Before checking, dismiss any visible cookie consent banners, "welcome back" dialogs, or notification prompts by looking for dismiss/close/decline buttons. If a CAPTCHA or bot-detection challenge appears, mark that platform as failed.

   - **Claude.ai** (`claude.ai`): Look for chat input area, user avatar, or model selector.
   - **Gemini** (`gemini.google.com`): Look for chat input area, user avatar.
   - **ChatGPT** (`chatgpt.com`): Look for message input box, user menu.
   - **Grok** (`grok.x.ai`): Look for chat input, user indicators.

3. Write `preflight.md`:
   ```markdown
   # Preflight Check — <timestamp>

   | Platform  | Status       | Notes |
   |-----------|-------------|-------|
   | Claude.ai | ready/failed | ...   |
   | Gemini    | ready/failed | ...   |
   | ChatGPT   | ready/failed | ...   |
   | Grok      | ready/failed | ...   |
   ```

4. **Decision point:**
   - If all platforms ready: proceed automatically.
   - If 1-2 platforms unavailable: tell the user which ones failed, ask if they want to proceed or fix logins first.
     - If the user says proceed: set the unavailable platforms' status to `"skipped"` in the manifest with notes explaining why. Increment `platforms_skipped`.
     - If the user says fix logins: pause and wait, then re-run preflight for those platforms.
   - If ALL platforms unavailable (Chrome disconnected): stop and tell the user to connect Chrome.

5. **Create the Notion row** (now that the research title exists from Step 2):
   - Topic: research_title
   - Status: "Running"
   - Claude / Gemini / ChatGPT / Grok: "Pending" (or "Skipped" if skipped in preflight)
   - Platforms: "0/4"
   - Started: current datetime
   - Session ID: the session folder name

**Important:** Preflight results are point-in-time. Execution steps must still handle login failures gracefully since session state can change between preflight and execution.

### Step 5: Execute Platform Queries

Run platforms **sequentially** in this order: Claude.ai, Gemini, ChatGPT, Grok.

Claude and Gemini are first because their advanced modes (Research / Deep Research) produce the highest-value responses — if the session is interrupted partway through, the most valuable responses are already captured.

**All 4 platforms are browser-automated.** Each gets a fresh conversation with just the research prompt. No platform sees another's output. Claude Code's role is orchestration and synthesis only.

**Execution rules that apply to ALL platforms:**
- Before each platform step, check if the platform's status is `"skipped"`. If so, skip it entirely.
- **Every execution step must re-navigate to its target URL.** Never assume the browser is still on the page left by preflight or a previous platform.
- Always start a fresh conversation. Never reuse an existing chat.
- Extract only the final assistant response, not the full page (no navigation elements, sidebars, or page chrome).
- **Error files** use the naming convention: `errors/<platform>_error.md` (e.g., `errors/claude_error.md`, `errors/chatgpt_error.md`). Update `error_file` in manifest.
- **Polling:** When waiting for a response, take a screenshot or use `get_page_text` every 45 seconds. Compare to the previous check to detect changes.
- **Prompt length:** After entering the prompt in the input field, verify the full text was accepted. If the input appears truncated, shorten the prompt by removing sub-questions from the research brief (last sub-question first), preserving the core question and scope. The full original prompt is already saved as `prompts/<platform>_prompt.md`. Save the shortened version that was actually submitted as `prompts/<platform>_submitted_prompt.md` (this file is only created when shortening was needed). Set `prompt_shortened: true` in manifest and note which sub-questions were removed.

**Stall detection and retry (same for all platforms):**
- If nothing on the page has changed for **5 consecutive minutes**, the platform is stalled.
- On first stall: set manifest notes to describe the stall, wait 30 seconds, refresh the page, start a fresh conversation, resubmit the same prompt (attempt #2).
- On second stall (retry also showed no activity for 5 min): mark as `"timed_out"`, save error details to `errors/<platform>_error.md`, take a screenshot (save path to `error_screenshot`), increment `platforms_timed_out`, continue to next platform.
- There is NO absolute timeout. If a platform is still actively generating (text appearing, progress updating, source count increasing), let it run as long as it needs — even over an hour.

**Hard failure handling (same for all platforms):**
- CAPTCHA, login error, crash, error page, or unexpected state: mark as `"failed"`, save error to `errors/<platform>_error.md`, take screenshot, increment `platforms_failed`, continue to next platform.

#### 5a: Claude.ai (Research Mode)

1. Update manifest: claude status -> `"running"`, set started_at, increment attempts.
2. In `automationTabId`, navigate to `claude.ai`.
3. Start a fresh conversation.
4. **Detect mode:** Look for a Research mode button, toggle, or option.
   - If found and activated: set mode_used -> `"research"`
   - If not found: use standard Claude, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt from `prompts/claude_prompt.md` and submit.
6. Wait for response (see polling and stall rules above).
7. Extract the final assistant response. Save to `responses/claude_response.md`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: Claude -> "Done", Platforms -> "1/4".
10. Tell the user: "Claude: done. Moving to Gemini..."

#### 5b: Gemini Deep Research

1. Update manifest: gemini status -> `"running"`, set started_at, increment attempts.
2. In `automationTabId`, navigate to `gemini.google.com`.
3. Start a fresh conversation.
4. **Detect mode:** Look for a Deep Research button, toggle, or option.
   - If found and activated: set mode_used -> `"deep_research"`
   - If not found: use standard Gemini, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt from `prompts/gemini_prompt.md` and submit.
6. Wait for response. Deep Research can run 30+ minutes or over an hour — this is normal if progress indicators are updating.
7. Extract the final assistant response. Save to `responses/gemini_response.md`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: Gemini -> "Done", Platforms -> update count.
10. Tell the user: "Gemini: done. Moving to ChatGPT..."

#### 5c: ChatGPT

1. Update manifest: chatgpt status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `automationTabId`, navigate to `chatgpt.com`.
3. Start a fresh conversation.
4. **Detect mode:** Look for ChatGPT Plus indicators (model selector showing GPT-4/GPT-4o, browsing toggle).
   - If Plus/browsing detected: set mode_used -> `"plus_with_browsing"`
   - If free tier detected: set mode_used -> `"free"`, add note.
5. Enter the rendered prompt from `prompts/chatgpt_prompt.md` and submit.
6. Wait for response (see polling and stall rules above).
7. Extract the final assistant response. Save to `responses/chatgpt_response.md`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: ChatGPT -> "Done", Platforms -> update count.
10. Tell the user: "ChatGPT: done. Moving to Grok..."

#### 5d: Grok

1. Update manifest: grok status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `automationTabId`, navigate to `grok.x.ai`.
3. Start a fresh conversation.
4. **Detect mode:** Look for Grok Premium or SuperGrok indicators.
   - If premium detected: set mode_used -> `"premium"`
   - If no premium indicators: set mode_used -> `"free"`
5. Enter the rendered prompt from `prompts/grok_prompt.md` and submit.
6. Wait for response (see polling and stall rules above).
7. Extract the final assistant response. Save to `responses/grok_response.md`.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion: Grok -> "Done", Platforms -> update count.
10. Tell the user: "Grok: done. All platforms complete. Synthesizing report..."

**After each platform failure or timeout**, also tell the user:
> "[Platform]: [failed/timed out] ([brief reason]). Moving to [next platform]..."

And update Notion for that platform accordingly (set to "Failed" or "Timed Out").

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

3. Add archive toggle to the Notion dashboard page (page ID `339e0acf-fc81-810e-bca1-c87aa4b960bd`) under the "Research Archive" section. The toggle content should be **inline text only** (Notion MCP cannot upload files):

   ```
   > [Research Title] (YYYY-MM-DD)
   >   > Executive Summary
   >     [paste executive summary text]
   >   > Confidence Assessment
   >     [paste confidence section text]
   >   > Platform Results
   >     Claude: [status] | Gemini: [status] | ChatGPT: [status] | Grok: [status]
   >   > Session Details
   >     Session ID: [id]
   >     Full responses and report stored locally in research/[session-id]/
   ```

4. Tell the user:
   - "Research complete! Report saved to `research/<session-id>/report.md`"
   - Brief summary of which platforms succeeded/failed/timed out
   - Note if any platform used a fallback mode (standard instead of Research/Deep Research)
   - Offer to show the executive summary

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
   - To update the existing Notion row: use the Notion search tool with the session ID as query and data source `878a53c0-7cb9-4f6f-8655-463d5d504337` as the `data_source_url`. If no row is found, create a new one.

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

The Notion dashboard provides live status tracking and a research archive.

**Database:** AI Research Sessions
- **Database ID:** `9d7f1ca4dc5d40b3b4638fe97c1992e7`
- **Data source ID:** `878a53c0-7cb9-4f6f-8655-463d5d504337`
- **Dashboard page ID:** `339e0acf-fc81-810e-bca1-c87aa4b960bd`

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
- **Session folders are self-contained.** Old sessions can be deleted without affecting anything.
- **Record what actually happened.** The manifest should accurately reflect reality — modes used, extractions, attempts, shortcuts taken.
