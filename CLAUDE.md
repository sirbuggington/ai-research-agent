# AI Research Agent — Orchestration Instructions

## What This Is

This is an automated multi-AI research system. When the user types a research request, you (Claude Code) orchestrate the entire workflow: generate tailored prompts, query 4 AI platforms via browser automation, collect responses, and synthesize a comprehensive report.

**Claude Code vs. Claude.ai:** In this workflow, Claude Code (the orchestrator) is a separate entity from Claude.ai (a research platform). Claude Code submits a prompt to Claude.ai via browser automation, just as it does for Gemini, ChatGPT, and Grok. Claude Code does not generate research content itself — it only orchestrates and synthesizes.

All file paths in this document are relative to the project root (the directory containing this CLAUDE.md file).

## Detail docs (load on demand)

This file is the top-level orchestrator. Detailed logic lives in separate files — read them with the `Read` tool when you reach the corresponding step:

- **`docs/platforms.md`** — Per-platform execution (Steps 5a-5d), input methods, extraction procedures, stall detection, depth-aware mode selection. Load this at Step 5.
- **`docs/setup.md`** — First-Time Setup questionnaire and task instructions. Load this when a setup trigger fires.
- **`docs/notion.md`** — Notion integration details. Load only if `config.local.json` -> `notion.enabled` is true.
- **`docs/evaluation.md`** — Self-evaluation scoring procedure, performance review, export performance report. Load at Step 7.5 and for the related commands.

Do NOT preemptively load all four. Read them only when you hit the step that needs them.

## Triggers

**Strict research trigger:** Only auto-run the full workflow when the user's message starts with:
- `Research: [topic]`
- `Research [topic]`

**Soft research trigger:** If the user says something like "research on [topic]", "look into [topic]", or "can you research [topic]" — ask for confirmation before launching the workflow:
> "Do you want me to run a full multi-AI research session on [topic]? This will query Claude, Gemini, ChatGPT, and Grok. Should I start?"

**Follow-up trigger:** If the user says `Follow up: <question>`, `Follow-up: <question>`, or `Drill down: <question>`, run the **Follow-up Session** flow (see section below).

**Resume trigger:** `Resume research` or `Continue research` — see Resume Rules below.

**Cancel trigger:** "stop", "cancel", or any unrelated message mid-run — see Cancellation Rules below.

**Setup trigger:** "setup", "set up", "get started", "let's get started", "how do I start", "how do I begin", "first time setup", "install", "initialize", or any clear equivalent — load `docs/setup.md` and run the questionnaire. Do NOT launch a research session.

**Performance review triggers:**
- `Review research performance` — print analysis to chat. See `docs/evaluation.md`.
- `Export performance report` — write a markdown file. See `docs/evaluation.md`.

Extract the topic/question from the user's message: everything after the trigger phrase.

## Local Configuration

This project reads runtime configuration from `config.local.json` at the project root. This file is gitignored. A template lives at `config.example.json`.

**At the start of every research session, follow-up, or setup run, read `config.local.json`.** If it does not exist, the user has not finished setup — direct them to the First-Time Setup flow.

Config keys:
- `chrome_profile_name` (string, required) — name of the Chrome profile signed into all 4 AI platforms.
- `notion.enabled` (bool) — if false or missing, **skip every Notion step in this workflow** (do not create rows, do not update statuses, do not append archive toggles). The workflow runs locally only.
- `notion.database_id`, `notion.data_source_id`, `notion.dashboard_page_id` — used only when `notion.enabled` is true.

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
| `skipped` | Deliberately skipped (unavailable at preflight or by depth setting) | User chose to proceed without this platform |
| `cancelled` | User interrupted the workflow | User cancelled mid-run |

**Session statuses (top-level in manifest):**
| Status | Meaning |
|--------|---------|
| `running` | Workflow in progress |
| `synthesizing` | All platforms done, report being generated |
| `verifying_sources` | Source verification in progress (Step 6.5, if opted in) |
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

4. **Current mode: recommendation-only.** Step 0 does NOT create adjustments in `adjustments.json`. It only logs recommendations and observations to `improvements.md` and shows the performance summary.

5. Check `learning/adjustments.json` — if `active_adjustment` is not null (manually created), increment `sessions_since_applied`. Otherwise, proceed.

### Step 1: Create Session

1. Generate a session ID: `YYYY-MM-DD-HHMM-topic-slug`
   - Use current date and time
   - Slugify the topic: lowercase, replace spaces with hyphens, remove special characters, truncate to 50 chars

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

3. Initialize `manifest.json` with:
   - `topic`, `research_title`, `raw_user_request`, `session_id`, `created_at`, `completed_at: null`, `status: "running"`
   - `depth: null`, `verify_sources: false` (set in Step 2.5)
   - `platforms` object — one entry per platform with `status: "pending"`, `mode_used: null`, `started_at: null`, `ended_at: null`, `attempts: 0`, `partial_extraction: false`, `prompt_shortened: false`, `prompt_file`, `response_file`, `error_file: null`, `error_screenshot: null`, `notes: ""`
   - Counters: `platforms_succeeded: 0`, `platforms_failed: 0`, `platforms_timed_out: 0`, `platforms_skipped: 0`
   - `report_file: "report.md"`

4. Tell the user: "Starting research session: `<session-id>`. Refining your topic..."

### Step 2: Refine the Research Topic

The user will provide a casual, conversational description of what they want researched (typically 2-8 sentences). Your job is to distill this into a structured research brief BEFORE it reaches any platform.

**Do NOT pass the user's raw input to the platforms verbatim.**

**Clarify, don't redirect.** Your job is to sharpen the user's question, not change it into a different question. If they asked about cat communication, the brief should be about cat communication — not animal cognition broadly. Stay faithful to what they actually want to know.

#### Topic narrowing guardrail

**Before generating the brief**, check whether the user's topic is too broad to research usefully. Apply the guardrail ONLY when all three are true:
- The topic is fewer than 5 words
- There are no proper nouns, dates, product names, or technical terms that pin a scope
- The topic would yield wildly different reports from different reasonable interpretations (e.g. "AI", "design", "climate")

If the guardrail triggers, call `AskUserQuestion` with ONE question before proceeding:

- Question: `"Your topic is very broad. Which angle should I focus on?"`
- header: `Angle`
- multiSelect: false
- options: propose 3 specific sub-angles that are reasonable interpretations of the user's topic. Each option label ≤5 words, description explains what the resulting research would cover. The user can always pick "Other" to supply a custom angle.

Use the user's answer to anchor the research brief. If they pick "Other" and supply text, treat that as the scope.

If the guardrail does NOT trigger (topic has specificity already), proceed directly to brief generation with no questionnaire.

#### Brief generation

1. Read the user's raw input. Identify:
   - The core question or topic they're actually asking about
   - Specific angles or sub-questions they mentioned (explicitly or implicitly)
   - Any scope boundaries (what they care about vs. don't care about)
   - The depth they seem to want (surface overview vs. deep dive)

2. Generate:
   - **Research Title** (short, clean — used for folder names, report headers)
   - **Research Brief** with: **Core question** (one sentence), **Key sub-questions** (3-6 items), **Scope** (in/out bounds), **Context** (any framing the user provided)

3. Save the research brief to `research/<session-id>/research_brief.md`. Update `manifest.json`: set `topic` to the core question and `research_title` to the generated title.

4. Show the user the research brief and title. Proceed to Step 2.5 immediately — do not wait for approval.

### Step 2.5: Depth and Source Verification

Before generating platform prompts, call `AskUserQuestion` ONCE with two questions:

1. **Question:** `"How deep should this research go?"`
   - header: `Depth`
   - multiSelect: false
   - options:
     - label: `High (Recommended)`, description: `"Full Deep Research / Research modes on all platforms. 30-60+ minutes. Most thorough."`
     - label: `Medium`, description: `"Claude Research + Grok Expert; ChatGPT and Gemini in standard mode. ~15 minutes. Balanced speed and depth."`
     - label: `Low`, description: `"All platforms in standard/fast mode. 3-5 minutes. Quick overview only."`

2. **Question:** `"Verify source URLs after research?"`
   - header: `Verify`
   - multiSelect: false
   - options:
     - label: `No`, description: `"Skip URL verification. Faster. Sources remain labeled as unverified."`
     - label: `Yes`, description: `"After synthesis, I'll fetch every cited URL, drop dead links, flag redirects. Adds 1-5 minutes."`

Store both answers in `manifest.json` (`depth` and `verify_sources`). These drive Step 5 (mode selection) and Step 6.5 (verification).

Then say: "Starting queries now — I'll update you as each platform completes."

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

3. For each platform, navigate its dedicated tab to the platform URL and check login state. Before checking, dismiss any visible cookie consent banners, "welcome back" dialogs, or notification prompts. If a CAPTCHA or bot-detection challenge appears, mark that platform as failed.

   - **Claude.ai** (`claude.ai`): Look for chat input area, user avatar, or model selector.
   - **Gemini** (`gemini.google.com`): Look for chat input area, user avatar.
   - **ChatGPT** (`chatgpt.com`): Look for message input box, user menu.
   - **Grok** (`grok.com`): Look for chat input, user indicators.

4. Write `preflight.md` (markdown table of Platform / Status / Notes).

5. **Decision point:**
   - If all platforms ready: proceed automatically.
   - If 1-2 platforms unavailable: tell the user which ones failed, ask if they want to proceed or fix logins first.
     - If the user says proceed: set the unavailable platforms' status to `"skipped"` in the manifest. Increment `platforms_skipped`.
     - If the user says fix logins: pause and wait, then re-run preflight for those platforms.
   - If ALL platforms unavailable (Chrome disconnected): stop and tell the user to connect Chrome.

6. **Create the Notion row** — *only if `config.local.json` -> `notion.enabled` is true. See `docs/notion.md` for details.*

**Important:** Preflight results are point-in-time. Execution steps must still handle login failures gracefully.

### Step 5: Execute Platform Queries

**Load `docs/platforms.md` now and follow its instructions.** That file contains:
- Execution rules that apply to all platforms
- Platform-specific input methods (critical — wrong method corrupts prompts)
- Mandatory prompt verification (before/after send)
- Parallel execution order
- Expected durations + stall detection + hard failure handling
- Periodic progress updates for the user (emit every ~5 min during polling)
- **Depth-aware mode selection** (uses `manifest.depth` from Step 2.5)
- Steps 5a through 5d (per-platform execution)

When all platforms finish (success, failed, timed out, or skipped), return here and proceed to Step 6.

### Step 6: Synthesize Report

1. Update manifest: status -> `"synthesizing"`. Update Notion (if enabled): Status -> "Synthesizing".
2. Read ALL response files from `research/<session-id>/responses/` (only those with status `"success"` in manifest).
3. Read the report template from `templates/report.md` and use it as a **structural guide** — follow its section headings and overall shape, but write naturally.
4. Write the report following this structure:

   **Title and metadata:** Research title, date, session ID, which platforms responded, which were unavailable and why, and the depth level used.

   **Executive Summary:** 2-3 paragraphs synthesizing the most important findings ACROSS all sources. Identify the overarching narrative — do not just summarize each AI separately.

   **Key Findings:** Organize by THEME, not by platform. Use 3-6 themes depending on topic complexity. Weave insights from multiple sources together under each theme.

   **Consensus & Disagreement:**
   - Agreement: where 2+ platforms reached similar conclusions (higher confidence)
   - Disagreement: where platforms contradicted each other (flag for verification)
   - Verification needed: claims from only one source that could not be confirmed

   **Unique Insights:** Things only one platform surfaced. Attribute clearly.

   **Sources & Citations:** Consolidate all URLs, paper references, and named sources. Label as **"Sources cited by platforms (not independently verified)."** — this label will be upgraded in Step 6.5 if the user opted in to verification.

   **Further Questions:** Follow-up research topics raised by this investigation.

   **Confidence Assessment:**
   - How many platforms responded out of 4
   - Depth level used (Low/Medium/High)
   - Which modes were used per platform
   - Whether any prompts were shortened or extractions were partial
   - Source quality and topic maturity assessment

   **Appendix:** Table showing each platform's final status and a note that full responses are in the local session folder.

5. Write the completed report to `research/<session-id>/report.md`.

### Step 6.5: Source Verification (conditional)

**Run only if `manifest.verify_sources` is true.** Otherwise, skip to Step 7.

1. Update manifest: status -> `"verifying_sources"`.
2. Tell the user: "Verifying sources — fetching each cited URL."
3. Extract every unique URL from the report's Sources section (and any URLs cited inline).
4. For each URL, use `WebFetch` to request it. Classify the result:
   - **alive**: 200 OK, same domain as URL
   - **redirected**: final URL differs from cited URL (note the destination)
   - **dead**: 4xx/5xx, DNS failure, or refused
   - **timeout**: no response within 15 seconds
5. Rewrite the Sources section of the report:
   - Group URLs by status: Alive / Redirected / Dead-or-unreachable
   - For redirected URLs, show both the cited URL and where it ended up
   - Remove dead URLs from the "Alive" bucket but keep them listed under "Dead-or-unreachable" with a note
6. Update the Confidence Assessment section: add a line `Source verification: X of Y URLs verified live (Z redirected, W dead).`
7. Change the Sources label from "not independently verified" to "verified live as of [date]".
8. Write the updated report back to `research/<session-id>/report.md`.

If WebFetch is unavailable or errors out for most URLs, log the failure in manifest notes, skip the rewrite, and continue. A missing verification pass is not a session failure.

### Step 7: Finalize

1. Update manifest.json:
   - Count `platforms_succeeded`, `platforms_failed`, `platforms_timed_out`, `platforms_skipped`
   - Set overall status:
     - `"completed"` — all 4 platforms succeeded
     - `"completed_with_errors"` — report generated but 1+ platforms failed/timed_out/skipped
     - `"failed"` — no platforms succeeded or report could not be generated
   - Set `completed_at` timestamp

2. Update Notion row (if enabled). See `docs/notion.md`.

3. **Clean up intermediate files:** Delete any `.docx`, `.txt`, `_raw.txt` artifacts from `Downloads/` and `responses/`. Only the final `.md` response files should remain.

4. Add archive toggle to the Notion dashboard page (if enabled). See `docs/notion.md` for the exact toggle format.

5. **Do NOT tell the user research is complete yet.** Proceed immediately to Step 7.5.

### Step 7.5: Self-Evaluation

**Load `docs/evaluation.md` now and follow its scoring procedure.** That file contains:
- The five scoring dimensions (Brief Fidelity, Response Relevance, Source Relevance, Synthesis Focus, Prompt Compliance)
- The post-scoring actions (write evaluation.json, append to ledger, update Notion with scores)
- The completion message to show the user

**This step is MANDATORY.** Do NOT ask the user whether to run it. Do NOT skip it. Do NOT report session completion until it finishes.

### Step 7.6: Export Question (after the completion message)

After showing the user the evaluation scores and "Research complete!" message, call `AskUserQuestion` ONCE:

- **Question:** `"Would you like me to export the report to a shareable format?"`
- header: `Export`
- multiSelect: false
- options:
  - label: `No thanks`, description: `"Keep it as markdown only"`
  - label: `PDF`, description: `"Convert report.md to a shareable PDF using the pdf skill"`
  - label: `DOCX`, description: `"Convert report.md to a Word document using the docx skill"`
  - label: `Both PDF and DOCX`, description: `"Produce both formats"`

If the user picks PDF or DOCX or Both:
1. Invoke the `anthropic-skills:pdf` skill (for PDF) or `anthropic-skills:docx` skill (for DOCX) with `research/<session-id>/report.md` as input.
2. Save the output(s) to the session folder as `report.pdf` and/or `report.docx`.
3. Tell the user: "Exported to `research/<session-id>/report.pdf`" (and/or `.docx`).

If the user picks "No thanks", say nothing further about export.

## Follow-up Sessions

**Trigger:** `Follow up: <question>`, `Follow-up: <question>`, or `Drill down: <question>`.

Follow-ups let the user drill into a specific point from a completed research session without spending another 30+ minutes running all 4 platforms. Follow-ups use Claude.ai only, in standard mode, for speed.

### Follow-up flow

1. **Identify the target session.** By default, use the **most recent** session folder in `research/` with status `completed` or `completed_with_errors`. If the user specifies a session ID in the form `Follow up (session <id>): <question>`, use that session instead. If no qualifying session exists, tell the user: "No completed session found. Start one with `Research: <topic>` first."

2. **Load context.** Read the target session's:
   - `report.md` (the synthesized report)
   - `research_brief.md` (original scope)
   - `manifest.json` (which platforms succeeded)
   - The `responses/` files from successful platforms

3. **Find or create the follow-up index.** Count existing `follow-up-*.md` files in the session folder. Use the next index (e.g., `follow-up-1.md`, `follow-up-2.md`).

4. **Open a Claude.ai tab.** Use existing `claudeTabId` if available; otherwise create a new tab via Tab Management. Navigate to `claude.ai` and start a fresh conversation. Do NOT activate Research mode — standard mode is faster and appropriate for a focused follow-up.

5. **Submit a follow-up prompt** that combines:
   - A brief one-paragraph summary of the original research (title + exec summary)
   - The user's follow-up question
   - A request for a focused, direct answer with any necessary context from the original scope

6. **Wait for response** using the same polling rules as Step 5 (5-min stall detection, retry once). Extract using the same two-step method (artifact panel + sources) documented in `docs/platforms.md` section 5a.

7. **Save the follow-up.** Write to `research/<session-id>/follow-up-<N>.md` with front matter:
   ```
   # Follow-up <N> — <YYYY-MM-DD>
   **Question:** <user's question>
   **Source session:** <session-id>

   ---

   <Claude's response>

   ## Sources
   <URLs>
   ```

8. **Append to manifest.** Add a `follow_ups` array to `manifest.json` (create if missing). Each entry: `{index, question, date, file, status}`.

9. **Tell the user:** "Follow-up saved to `research/<session-id>/follow-up-<N>.md`." Briefly show the answer's key point in chat.

Follow-ups are NOT evaluated by Step 7.5 (they're supplemental, not primary research).

## Cancellation Rules

If the user interrupts a running research session (says "stop", "cancel", or sends an unrelated message):

1. Stop the current platform operation immediately.
2. Update the manifest:
   - Set any platform currently `"running"` to `"cancelled"`.
   - Leave `"success"` platforms unchanged.
   - Leave `"pending"` platforms as `"pending"`.
   - Set session status to `"cancelled"`.
3. Update Notion (if enabled): set the running platform to "Cancelled", set session Status to "Cancelled".
4. Tell the user:
   > "Research session cancelled. [N] platforms completed before cancellation. You can resume later with 'Resume research'."
5. Do NOT delete any files. Partial sessions can be resumed.

## Resume Rules

When the user says "Resume research" or "Continue research":

1. Scan `research/` for all session folders. Read each `manifest.json`.
2. Find sessions with status `"running"`, `"failed"`, `"completed_with_errors"`, `"cancelled"`, or `"verifying_sources"`.
3. **If no resumable sessions exist:** tell the user all sessions are complete and offer to start a new one.
4. **If exactly one resumable session exists:** resume it automatically.
5. **If multiple resumable sessions exist:** list them and ask the user which one to resume.

6. To resume a session:
   - Set session status back to `"running"` (or `"verifying_sources"` if that was where it stopped).
   - Identify which platforms have status `"pending"`, `"failed"`, `"timed_out"`, or `"cancelled"`.
   - Platforms with status `"skipped"` are NOT retried (the user already chose to skip them).
   - Platforms with status `"success"` are NOT re-run.
   - Run preflight for the platforms that need retrying.
   - Execute only those platforms.
   - After all retries complete, re-synthesize the report with all available responses.
   - If `verify_sources` was true and verification hadn't completed, run Step 6.5 now.
   - Update manifest and Notion.
   - To update the existing Notion row (only if enabled): see `docs/notion.md`.

## Stall Detection Reference

All browser platforms use the same rule: **5 minutes of no visible change = stalled.** Full detail in `docs/platforms.md`.

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

## Important Notes

- **All platforms are independent.** Each gets a fresh conversation with just the research prompt. No platform sees another's output.
- **Always re-navigate.** Every execution step navigates fresh to its target URL. Never assume the browser is on any particular page.
- **Always start fresh conversations.** Never reuse an existing chat on any platform.
- **Do not rush responses.** Wait for each platform to fully complete. Partial responses are worse than waiting.
- **Keep the user informed.** Status updates after every platform completes or fails, plus periodic progress lines every ~5 min during long polls (see `docs/platforms.md`).
- **The report is the product.** Spend real effort on synthesis. Raw responses are reference material.
- **Sources are unverified by default.** AI platforms hallucinate URLs. Unless Step 6.5 ran, label source lists as unverified.
- **Every response must include sources.** After saving each response file, verify it contains a sources/citations section with actual URLs. If sources are missing, extract them from the platform's DOM before proceeding.
- **Session folders are self-contained.** Old sessions can be deleted without affecting anything.
- **Record what actually happened.** The manifest should accurately reflect reality — modes used, depth level, extractions, attempts, shortcuts taken.
- **Never use clipboard for extraction.** `navigator.clipboard.readText()` fails when the tab is not focused. Always use downloads or direct DOM extraction.
- **Never delegate browser automation to background agents.** Background agents may not receive permissions for browser MCP tools. Perform all browser operations inline in the main conversation.
- **Click small UI elements by ref, not coordinates.** Use `read_page` to find the exact element reference first, then click by ref.
- **Manage context aggressively.** Browser automation consumes large amounts of context. Once a platform response is extracted and saved, do not re-read the full response unless needed for synthesis. Minimize screenshot frequency.
- **Mid-session context recovery.** If the context window fills up and the session continues in a new conversation, reconstruct state by: (1) reading `manifest.json` for platform statuses and session metadata, (2) checking which response files exist in `responses/`, (3) resuming from the first platform that isn't `"success"`. The manifest is the source of truth.
- **User interaction during research.** The user may interact with the browser while research is running (browsing other tabs, etc.). This is generally safe. Switching to or closing one of the 4 platform tabs can break polling or extraction. If a platform tab becomes unresponsive or navigates away, create a new tab and restart that platform.
