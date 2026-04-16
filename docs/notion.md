# Notion Integration

**Load this doc only when `config.local.json` -> `notion.enabled` is true.** When Notion is disabled, skip every step here and every Notion-related action elsewhere in the workflow.

When enabled, the dashboard provides live status tracking and a research archive.

## Database info

- **Database name:** the user's "AI Research Sessions" database
- **Database ID:** read from `config.local.json` -> `notion.database_id`
- **Data source ID:** read from `config.local.json` -> `notion.data_source_id`
- **Dashboard page ID:** read from `config.local.json` -> `notion.dashboard_page_id`

## Verified Notion property names (exact match required)

`Topic` (title), `Status` (select), `Claude` (select), `Gemini` (select), `ChatGPT` (select), `Grok` (select), `Platforms` (text), `Tags` (multi-select), `Started` (date), `Completed` (date), `Session ID` (text)

## Notion display values

- **Platform columns** (Claude / Gemini / ChatGPT / Grok): `Pending`, `Running`, `Done`, `Failed`, `Timed Out`, `Skipped`, `Cancelled`
- **Status column:** `Running`, `Synthesizing`, `Completed`, `Completed with Errors`, `Failed`, `Cancelled`

## When to update Notion

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

## Archive toggle format

Use `<details>/<summary>` HTML tags for Notion toggles. Content inside must be **tab-indented**. `<summary>` must be all on ONE line.

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

## Finding an existing row (for resume/update)

Use the Notion search tool with the session ID as query and the `data_source_id` from `config.local.json` as the `data_source_url`. If no row is found, create a new one.

## Notion update failures

If a Notion update fails (API error, rate limit), do NOT stop the research workflow. Log the failure in manifest notes and continue. Local files are the source of truth. Retry failed Notion updates during Step 7.
