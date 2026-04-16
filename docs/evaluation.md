# Self-Evaluation and Performance Review

**Load this doc for Step 7.5 of the workflow and when the user runs performance-review commands.**

## Step 7.5: Self-Evaluation (Intent Tracing)

**This step is MANDATORY and runs automatically.** Do NOT ask the user whether to run it. Do NOT skip it. Do NOT report session completion until it finishes.

After the report is complete, perform an automated self-evaluation. This traces the user's original intent through the entire pipeline and scores how well it was preserved at each stage.

**Inputs:** Read `raw_user_request` from manifest, `research_brief.md`, all response files (only for platforms with status `"success"`), and `report.md`.

### Scoring Procedure

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

### After Scoring

1. Compute prompt hashes: `cksum prompts/system.md | cut -d' ' -f1` (truncated to 8 chars). Same for each platform template.
2. Auto-classify topic category from manifest topic: tech, design, science, health, business, policy, culture, other.
3. Compute execution metrics from manifest (duration, platform counts, retry count, context exhaustion).
4. Compute per-platform metrics from response files (response_length_chars, source_count, duration_minutes).
5. Write `evaluation.json` to the session folder following the schema in `templates/evaluation_schema.json`. Every score must include its required evidence (the mappings, checklists, and classifications described above).
6. Determine `low_confidence`: true if fewer than 2 platforms succeeded.
7. Append a compressed entry to `learning/ledger.json` (scores, execution metrics, platform summary, prompt hashes, topic category, low_confidence flag).
8. Update the Notion row (if enabled) with: Brief Fidelity, Synthesis Focus, Duration, Eval Summary (compact per-platform string).
9. Add `"evaluation_file": "evaluation.json"` to manifest.json.
10. **NOW tell the user the session is complete:**
    - "Research complete! Report saved to `research/<session-id>/report.md`"
    - Brief summary of which platforms succeeded/failed/timed out
    - Note if any platform used a fallback mode (standard instead of Research/Deep Research)
    - Show the evaluation scores (Brief Fidelity, Synthesis Focus, per-platform Response Relevance)
    - Offer to show the executive summary

## Review Research Performance

**Trigger:** When the user says `Review research performance`:

1. Read `learning/ledger.json`.
2. If fewer than 3 entries: "Not enough data yet (N sessions). Need at least 3 for meaningful analysis."
3. If 3+ entries: generate a full analysis in chat:
   - Score trends over all sessions (text table)
   - Per-platform performance comparison (average scores, rankings)
   - Category-specific breakdowns (where 3+ sessions exist)
   - Prompt hash change detection (which templates changed between sessions)
   - Weakest dimensions with diagnosis
   - Recommendations from `improvements.md`
   - Adjustment history (if any exist in `adjustments.json`)

## Export Performance Report

**Trigger:** When the user says `Export performance report`:

Generate a human-readable markdown file instead of printing to chat. This is useful for sharing, archiving, or reading in a file viewer.

1. Read `learning/ledger.json`.
2. If fewer than 3 entries: print "Not enough data yet (N sessions). Need at least 3 to export a meaningful report."
3. Otherwise, generate a markdown file at `learning/performance-report-YYYY-MM-DD.md` with these sections:
   - **Summary** — total sessions, date range covered, overall averages for Brief Fidelity and Synthesis Focus.
   - **Per-platform averages** — table with columns: Platform, Response Relevance avg, Source Relevance avg, Prompt Compliance avg, Success Rate. Sort by Response Relevance descending.
   - **Category breakdown** — for each topic category with 3+ sessions: averages + session count.
   - **Trend** — a compact text chart of Brief Fidelity and Synthesis Focus scores session-by-session, most recent last.
   - **Weakest dimensions** — identify the lowest-scoring dimension overall and the lowest per platform. Include diagnosis notes from the ledger or improvements.md.
   - **Active recommendations** — paste current "Pending review" entries from `improvements.md`.
   - **Session index** — table with session_id, date, topic_short, platforms_succeeded/4, Brief Fidelity, Synthesis Focus.
4. Tell the user: "Performance report written to `learning/performance-report-YYYY-MM-DD.md`."
