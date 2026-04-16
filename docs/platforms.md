# Per-Platform Execution Details

**Load this document when executing Step 5 of the workflow.** It contains every platform-specific quirk, input method, and extraction technique. Keeping this out of CLAUDE.md keeps the main orchestration doc lean.

## Execution rules that apply to ALL platforms

- Before each platform step, check if the platform's status is `"skipped"`. If so, skip it entirely.
- **Every execution step must re-navigate to its target URL.** Never assume the browser is still on the page left by preflight.
- Always start a fresh conversation. Never reuse an existing chat.
- Extract only the final assistant response, not the full page (no navigation elements, sidebars, or page chrome).
- **Every response file must include source URLs.** After extraction, verify that sources are present. If they're missing, extract them from the platform's DOM before moving on.
- **Error files** use the naming convention: `errors/<platform>_error.md` (e.g., `errors/claude_error.md`, `errors/chatgpt_error.md`). Update `error_file` in manifest.
- **Polling (parallel):** After submitting prompts to all 4 platforms, cycle through each tab checking for completion. Use `get_page_text` or a quick screenshot on each tab. Check each tab roughly every 60-90 seconds. A platform is "done" when the response is fully rendered, any stop/generating button is gone, and no new text is appearing. Start extraction on each platform as it completes — don't wait for all 4.
- **Prompt length:** After entering the prompt in the input field, verify the full text was accepted. If the input appears truncated, shorten the prompt by removing sub-questions from the research brief (last sub-question first), preserving the core question and scope. The full original prompt is already saved as `prompts/<platform>_prompt.md`. Save the shortened version that was actually submitted as `prompts/<platform>_submitted_prompt.md` (this file is only created when shortening was needed). Set `prompt_shortened: true` in manifest and note which sub-questions were removed.

## Platform-specific input methods (CRITICAL — using the wrong method corrupts prompts)

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

## Parallel execution order

1. Navigate all 4 tabs to their platform URLs and start fresh conversations.
2. Activate each platform's research/expert mode and submit prompts to all 4.
3. Poll all tabs in rotation, extracting each response as it completes.
4. Update manifest and Notion after each platform finishes.

## Expected research durations (for stall calibration)

| Platform | Typical Duration | Notes |
|----------|-----------------|-------|
| Grok Expert | 5-15 minutes | Fastest; single thinking phase |
| Claude Research | 8-10 minutes | Shows source count increasing |
| ChatGPT Deep Research | 20-30 minutes | Shows research progress steps |
| Gemini Deep Research | 30-60+ minutes | Longest; shows "Sources found" count |

Do not flag a platform as stalled just because it's taking longer than expected. Only flag based on the 5-minute no-change rule below.

## Progress updates during polling

To keep the user informed during long waits, emit a one-line status update in the chat **every ~5 minutes** while polling. Format:

> "Still running — Claude: 12 sources found | ChatGPT: analyzing vendor landscape | Gemini: 43 sources | Grok: done"

Pull the snippets from each tab's current visible state (last progress indicator, source count, current step label). Skip platforms with status `"success"`, `"failed"`, `"timed_out"`, or `"skipped"` (just show their final state or omit). Keep the whole line to one sentence — do not expand across multiple lines.

## Stall detection and retry (same for all platforms)

- If nothing on the page has changed for **5 consecutive minutes**, the platform is stalled.
- On first stall: set manifest notes to describe the stall, wait 30 seconds, refresh the page, start a fresh conversation, resubmit the same prompt (attempt #2).
- On second stall (retry also showed no activity for 5 min): mark as `"timed_out"`, save error details to `errors/<platform>_error.md`, take a screenshot (save path to `error_screenshot`), increment `platforms_timed_out`, continue to next platform.
- There is NO absolute timeout. If a platform is still actively generating (text appearing, progress updating, source count increasing), let it run as long as it needs — even over an hour.

## Hard failure handling (same for all platforms)

- CAPTCHA, login error, crash, error page, or unexpected state: mark as `"failed"`, save error to `errors/<platform>_error.md`, take screenshot, increment `platforms_failed`, continue to next platform.

## Depth-aware mode selection

The user selects a research depth during session setup (Low / Medium / High — see CLAUDE.md Step 2.5). Use this table to pick which mode to activate per platform:

| Depth | Claude | Gemini | ChatGPT | Grok |
|-------|--------|--------|---------|------|
| Low | standard | standard | standard | standard (NOT Expert) |
| Medium | Research | standard | standard | Expert |
| High (default) | Research | Deep Research | Deep Research | Expert |

Follow the per-platform "Activate X mode" steps below only when that mode is selected. For lower depths, skip the mode-activation sub-steps and submit directly in the default (standard) mode. Still set `mode_used` in the manifest accordingly.

## 5a: Claude.ai

1. Update manifest: claude status -> `"running"`, set started_at, increment attempts.
2. In `claudeTabId`, navigate to `claude.ai`.
3. Start a fresh conversation.
4. **Activate Research mode** (only if depth is Medium or High):
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
   c. Click "Copy options" ref, then click "Download as Markdown" from the dropdown.
   d. Read the downloaded `.md` file from `Downloads/` folder.
   e. The markdown has the full report text BUT source citation badges are plain text without URLs. Extract source URLs from the artifact DOM: call `read_page` on the artifact panel content area ref, then collect all `<a href>` elements — these are the inline citation links.
   f. Deduplicate the URLs and append a `## Sources` section to the response file.
   g. Save the final file to `responses/claude_response.md`.
   h. Delete the downloaded markdown from `Downloads/` (intermediate artifact).
9. Update manifest: status -> `"success"`, set ended_at.
10. Update Notion (if enabled): Claude -> "Done", Platforms count updated.
11. Tell the user: "Claude: done."

## 5b: Gemini

1. Update manifest: gemini status -> `"running"`, set started_at, increment attempts.
2. In `geminiTabId`, navigate to `gemini.google.com`.
3. Start a fresh conversation.
4. **Activate Deep Research mode** (only if depth is High):
   - Ensure "Thinking" is selected in the mode dropdown (right side of input)
   - Click the "Tools" button (to the right of the `+` button)
   - Click "Deep research" from the tools menu
   - A "Deep research" badge should appear in the input area confirming activation
   - Set mode_used -> `"deep_research"`
   - If "Deep research" is not in the tools menu: use standard Gemini, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt using the `type` action (click input ref first, then type). Do NOT use JavaScript injection — Gemini's TrustedHTML policy blocks `innerHTML` and `execCommand` fails silently. Verify full text is present before submitting.
6. Click the send button. **If the page refreshes instead of processing:** immediately check network requests via `read_network_requests` for a 503 status on the `StreamGenerate` endpoint. If 503 is present, this is server-side rate limiting — do NOT waste time trying different click methods (ref clicks, coordinate clicks, JS clicks, keyboard shortcuts all fail equally). Mark Gemini as `"failed"` and move on. The 503 can be triggered by rapid automated page loads and JS manipulation.
7. **Gemini will present a research plan (Deep Research only).** Click "Start" or the equivalent approval button to begin the deep research. Wait for research to complete — this can run 30+ minutes or over an hour. This is normal if progress indicators are updating.
8. **Extract response — Gemini requires JavaScript extraction:**
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
9. Update manifest: status -> `"success"`, set ended_at.
10. Update Notion (if enabled): Gemini -> "Done", Platforms count updated.
11. Tell the user: "Gemini: done."

## 5c: ChatGPT

1. Update manifest: chatgpt status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `chatgptTabId`, navigate to `chatgpt.com`.
3. Start a fresh conversation.
4. **Activate Deep Research mode** (only if depth is High):
   - Click the `+` button in the input area
   - Click "Deep research" from the menu
   - A "Deep research" badge should appear confirming activation
   - The input placeholder should change to "Get a detailed report"
   - Set mode_used -> `"deep_research"`
   - If "Deep research" is not in the menu: use standard ChatGPT, set mode_used -> `"standard"`, add note.
5. Enter the rendered prompt from `prompts/chatgpt_prompt.md` and submit.
6. **ChatGPT may present a research plan to approve (Deep Research only).** Click the approval/start button to begin. Wait for response (see polling and stall rules above).
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
9. Update Notion (if enabled): ChatGPT -> "Done", Platforms count updated.
10. Tell the user: "ChatGPT: done."

## 5d: Grok

1. Update manifest: grok status -> `"running"`, set started_at, increment attempts. Do NOT set mode_used yet.
2. In `grokTabId`, navigate to `grok.com`.
3. Start a fresh conversation.
4. **Set mode:**
   - Click the mode selector dropdown (shows current mode like "Expert" or "Fast").
   - For depth Medium or High: select "Expert" (Thinks hard - Grok 4.20). Set mode_used -> `"expert"`.
   - For depth Low: leave in default fast mode. Set mode_used -> `"standard"`.
   - Grok does not have a separate research mode — it will do research when prompted to do so.
5. Enter the rendered prompt from `prompts/grok_prompt.md` and submit.
6. Wait for response (see polling and stall rules above).
7. **Extract response — Grok is the simplest extraction:**
   a. Use `get_page_text` — this works cleanly for Grok and captures the full response within the character limit.
   b. Extract additional source URLs via JavaScript from the Sources panel/sidebar if present.
   c. Save the response text with all sources appended to `responses/grok_response.md`.
   d. Note: Grok reports "N sources" in its header (e.g., "68 sources") but typically only lists 8-10 explicit URLs. The remaining sources were used during thinking but not surfaced. This is a Grok limitation, not an extraction failure.
8. Update manifest: status -> `"success"`, set ended_at.
9. Update Notion (if enabled): Grok -> "Done", Platforms count updated.
10. Tell the user: "Grok: done."

## After each platform

Tell the user:
> "[Platform]: [done/failed/timed out] ([brief reason if applicable])."

And update Notion for that platform accordingly (if enabled).

When all platforms are done (or failed/timed out), tell the user:
> "All platforms complete. Synthesizing report..."
