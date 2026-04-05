# AI Research Agent — Orchestration Instructions

## What This Is

This is an automated multi-AI research system. When the user types a research request, you (Claude Code) orchestrate the entire workflow: generate tailored prompts, query 4 AI platforms, collect responses, and synthesize a comprehensive report.

## Trigger

Recognize any of these as a research request:
- `Research: [topic]`
- `Research [topic]`
- `research on [topic]`
- `look into [topic]`
- Or any message that clearly asks you to research a topic using the multi-AI system

Extract the topic from the user's message. The topic is everything after the trigger phrase.

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
   ├── preflight.md
   ├── prompts/
   ├── responses/
   └── errors/
   ```

3. Initialize `manifest.json`:
   ```json
   {
     "topic": "<the research topic>",
     "session_id": "<session-id>",
     "created_at": "<ISO timestamp>",
     "status": "running",
     "platforms": {
       "gemini": { "status": "pending", "started_at": null, "ended_at": null, "prompt_file": "prompts/gemini_prompt.md", "response_file": "responses/gemini_response.md", "error_file": null, "notes": "" },
       "chatgpt": { "status": "pending", "started_at": null, "ended_at": null, "prompt_file": "prompts/chatgpt_prompt.md", "response_file": "responses/chatgpt_response.md", "error_file": null, "notes": "" },
       "grok": { "status": "pending", "started_at": null, "ended_at": null, "prompt_file": "prompts/grok_prompt.md", "response_file": "responses/grok_response.md", "error_file": null, "notes": "" },
       "claude": { "status": "pending", "started_at": null, "ended_at": null, "prompt_file": "prompts/claude_prompt.md", "response_file": "responses/claude_response.md", "error_file": null, "notes": "" }
     },
     "platforms_succeeded": 0,
     "platforms_failed": 0,
     "report_file": "report.md"
   }
   ```

4. Tell the user: "Starting research session: `<session-id>`. Running preflight checks..."

### Step 2: Generate Prompts

1. Read `prompts/system.md` — this is the shared context
2. Read each platform-specific template: `prompts/claude.md`, `prompts/gemini.md`, `prompts/chatgpt.md`, `prompts/grok.md`
3. For each template:
   - Replace `{{SYSTEM_PROMPT}}` with the contents of `system.md` (with `{{TOPIC}}` already replaced)
   - Replace `{{TOPIC}}` with the user's research topic
4. Save rendered prompts to the session folder:
   - `research/<session-id>/prompts/claude_prompt.md`
   - `research/<session-id>/prompts/gemini_prompt.md`
   - `research/<session-id>/prompts/chatgpt_prompt.md`
   - `research/<session-id>/prompts/grok_prompt.md`

### Step 3: Preflight Check

Check each platform's availability and save results to `preflight.md`.

1. **Chrome MCP connection**: Call `tabs_context_mcp` with `createIfEmpty: true`. If this fails, browser platforms are unavailable.

2. **Gemini**: Navigate a tab to `gemini.google.com`. Check if the page shows a logged-in state (look for a chat input area, user avatar, or similar). Record: ready / not logged in / unreachable.

3. **ChatGPT**: Navigate a tab to `chatgpt.com`. Check for logged-in state (message input box, user menu). Record: ready / not logged in / unreachable.

4. **Grok**: Navigate a tab to `grok.x.ai`. Check for logged-in state (chat input, user indicators). Record: ready / not logged in / unreachable.

5. **Claude**: Always ready (native).

Write `preflight.md`:
```markdown
# Preflight Check — <timestamp>

| Platform | Status | Notes |
|----------|--------|-------|
| Gemini   | ready/failed | ... |
| ChatGPT  | ready/failed | ... |
| Grok     | ready/failed | ... |
| Claude   | ready | Native — always available |
```

**Decision point:**
- If all platforms ready: proceed automatically
- If 1-2 platforms unavailable: tell the user which ones failed, ask if they want to proceed with what's available or fix logins first
- If ALL browser platforms unavailable (Chrome disconnected): stop and tell the user to connect Chrome

### Step 4: Execute Platform Queries

Run platforms **sequentially** in this order. This order is intentional — browser work first, native last.

#### 4a: Gemini Deep Research

1. Update manifest: gemini status → "running", set started_at
2. Navigate to `gemini.google.com`
3. Start a new chat if needed (look for "New chat" or equivalent)
4. **Important**: Look for and activate Deep Research mode if available (it may be a button, toggle, or option in the interface). If Deep Research mode is not available or you cannot find it, proceed with a regular Gemini query.
5. Find the message input field
6. Read the rendered prompt from `research/<session-id>/prompts/gemini_prompt.md`
7. Enter the prompt into the input field and submit
8. **Wait for response to complete:**
   - Monitor the page for signs that generation is still in progress (loading spinners, streaming text, "thinking" indicators)
   - Check every 15-20 seconds
   - **Timeout: 5 minutes** for Deep Research (it can take a while)
   - If Deep Research is running, you may see a multi-step progress indicator — wait for it to complete fully
9. Once complete, extract the full response text using `get_page_text` or `read_page`
10. Save response to `research/<session-id>/responses/gemini_response.md`
11. Update manifest: status → "success", set ended_at

**On failure:**
- Save error details to `research/<session-id>/errors/gemini_error.md`
- Update manifest: status → "failed" or "timed_out", set notes
- Continue to next platform

#### 4b: ChatGPT

1. Update manifest: chatgpt status → "running", set started_at
2. Navigate to `chatgpt.com`
3. Start a new chat (click "New chat" button or navigate to the main page)
4. Find the message input field
5. Read the rendered prompt from `research/<session-id>/prompts/chatgpt_prompt.md`
6. Enter the prompt and submit
7. **Wait for response to complete:**
   - Watch for the streaming to finish (stop button disappears, response is fully rendered)
   - Check every 10-15 seconds
   - **Timeout: 3 minutes**
8. Extract full response text
9. Save to `research/<session-id>/responses/chatgpt_response.md`
10. Update manifest: status → "success", set ended_at

**On failure:** Same pattern — save error, update manifest, continue.

#### 4c: Grok

1. Update manifest: grok status → "running", set started_at
2. Navigate to `grok.x.ai`
3. Start a new conversation if needed
4. Find the message input field
5. Read the rendered prompt from `research/<session-id>/prompts/grok_prompt.md`
6. Enter the prompt and submit
7. **Wait for response to complete:**
   - Watch for streaming to finish
   - Check every 10-15 seconds
   - **Timeout: 2 minutes**
8. Extract full response text
9. Save to `research/<session-id>/responses/grok_response.md`
10. Update manifest: status → "success", set ended_at

**On failure:** Same pattern.

#### 4d: Claude (Native)

1. Update manifest: claude status → "running", set started_at
2. Read the rendered prompt from `research/<session-id>/prompts/claude_prompt.md`
3. Generate a thorough, comprehensive response to the prompt. This is YOUR analysis — make it count. You have all the context and capabilities of Claude. Write the kind of deep, nuanced analysis that is your strength.
4. Save your response to `research/<session-id>/responses/claude_response.md`
5. Update manifest: status → "success", set ended_at

### Step 5: Synthesize Report

1. Read ALL successful response files from `research/<session-id>/responses/`
2. Read the report template from `templates/report.md`
3. Generate the synthesis report by filling in the template structure:

   **Executive Summary:** 2-3 paragraphs synthesizing the most important findings ACROSS all sources. Do not just summarize each AI separately — identify the overarching narrative.

   **Key Findings:** Organize by THEME, not by platform. If Claude said X about a topic and Gemini found supporting evidence for X, weave those together under one theme heading. Use 3-6 themes depending on topic complexity.

   **Consensus & Disagreement:**
   - Agreement: where 2+ platforms reached similar conclusions (higher confidence)
   - Disagreement: where platforms contradicted each other (flag for verification)
   - Verification needed: claims from only one source that could not be confirmed

   **Unique Insights:** Things only one platform surfaced. Attribute clearly: "Grok noted that..." or "Gemini found a source indicating..."

   **Sources & Citations:** Consolidate all URLs, paper references, and named sources from all responses into one list.

   **Further Questions:** What follow-up research would deepen understanding?

   **Confidence Assessment:**
   - Note how many platforms responded
   - If fewer than 4: explicitly note reduced confidence
   - Assess source quality (were citations provided? were sources authoritative?)
   - Assess topic maturity (well-established field vs. emerging/evolving topic)

4. Write the completed report to `research/<session-id>/report.md`

### Step 6: Finalize

1. Update manifest.json:
   - Set overall status to "completed"
   - Count platforms_succeeded and platforms_failed
   - Set completed_at timestamp

2. Tell the user:
   - "Research complete! Report saved to `research/<session-id>/report.md`"
   - Brief summary of which platforms succeeded/failed
   - Offer to show the executive summary or open the full report

## Platform Timeout Reference

| Platform | Timeout | Reason |
|----------|---------|--------|
| Gemini Deep Research | 5 minutes | Deep Research can run extended searches |
| ChatGPT | 3 minutes | Web browsing + generation |
| Grok | 2 minutes | Generally fast responses |
| Claude | No timeout | Native — runs directly |

## Error Handling Rules

1. **One platform fails:** Skip it, note in manifest and errors/ folder, continue with the rest. The report clearly states which platforms contributed.

2. **Multiple platforms fail:** Still produce a report with whatever responded, but add a prominent "Limited Confidence" warning. If only Claude responded, note that the report lacks web-sourced or real-time information.

3. **Chrome disconnected mid-run:** Save whatever has been collected so far. Tell the user what happened and what was saved. They can reconnect Chrome and you can attempt the remaining platforms.

4. **Unexpected page state:** If a platform shows an unexpected page (CAPTCHA, maintenance, error page, popup), take a screenshot, save it as the error file, mark the platform as failed, and continue.

5. **Response extraction fails:** If you can't cleanly extract the response text, save whatever you can get and note "partial extraction" in the manifest.

## Important Notes

- **Do not rush responses.** Wait for each platform to fully complete before extracting. Partial responses are worse than waiting an extra 30 seconds.
- **Keep the user informed.** Provide brief status updates as each platform completes: "Gemini: done. Moving to ChatGPT..."
- **The report is the product.** Spend real effort on the synthesis. The raw responses are reference material — the report is what the user will read and rely on.
- **Session folders are self-contained.** Everything about a research run lives in its folder. Old sessions can be deleted without affecting anything.
