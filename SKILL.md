---
name: summarizing-contributions
description: Summarizes a person's work contributions over a time period into a themed, impact-focused report from their merged GitHub PRs, resolved Jira tickets, and created Confluence pages. Use when the user asks to summarize their contributions, build a self-review or brag doc, or recap what they shipped in a period.
disable-model-invocation: true
---

# Summarizing Contributions

Produce a themed, impact-focused summary of the user's work over a time period,
drawn from their merged GitHub PRs, resolved Jira tickets, and created Confluence
pages. The output is a single local markdown file. Read `reference.md` for the exact
identity resolution, date math, query recipes, and output template before gathering data.

## Workflow

Copy this checklist and track progress:

```
- [ ] Step 1: Resolve the time period to concrete dates and confirm with the user
- [ ] Step 2: Resolve identity (GitHub login; Jira/Confluence use currentUser())
- [ ] Step 3: Gather items from GitHub, Jira, Confluence
- [ ] Step 4: Enrich items by reading descriptions
- [ ] Step 5: Synthesize 3-6 themes
- [ ] Step 6: Write the markdown file and print its path
```

### Step 1: Resolve the period

Accept either natural language (`Q1 2026`, `last quarter`, `May 2026`, `last 30 days`)
or explicit `YYYY-MM-DD` start/end dates. Convert to concrete `start` and `end` dates
using the rules in `reference.md` ("Date-range math"). State the resolved range back to
the user (e.g. "Summarizing 2026-01-01 to 2026-03-31 — proceeding") before querying.

### Step 2: Resolve identity

- GitHub: call `get_me` and keep the `login` field.
- Jira and Confluence: no lookup needed — the queries use `currentUser()`.
- Optionally call `jira_get_user_profile` with the user's email to show their display
  name in the report. If GitHub `get_me` fails, ask the user for their GitHub username.

### Step 3: Gather

Run the three queries exactly as written in `reference.md` ("Query recipes"),
substituting the resolved `start`, `end`, and GitHub `login`. Page through results until
exhausted. Keep, per item: title, identifier, URL, and the relevant date.

### Step 4: Enrich

For each gathered item, read its description to understand impact (medium depth):
- GitHub: `pull_request_read` for the PR body.
- Jira: `jira_get_issue` for the description.
- Confluence: `confluence_get_page` for the page summary.
Enrich every item when there are <= 40 total; above that, enrich only items whose titles
look substantial and summarize the rest under "By the numbers".

### Step 5: Synthesize themes

Cluster the enriched items into 3-6 impact areas (e.g. "Reliability & Performance",
"New features", "Developer experience", "Documentation"). For each theme write a short
narrative of what was done and why it mattered, with the underlying items as linked
evidence. Every claim must trace to a real linked item — never invent contributions.

### Step 6: Write output

Render the template in `reference.md` ("Output template") and write it to
`~/contributions/contributions-<period>.md` (create `~/contributions/` if missing; use a
slug like `2026-Q1` or `2026-01-01_2026-03-31` for `<period>`). Print the final file path.

## Edge cases

- A source returns nothing or identity can't be resolved there: note it and continue.
- Empty period overall: state that nothing was found; do not fabricate content.
- Always evidence-based: each theme bullet links to a real PR, ticket, or page.
- A source server is unavailable mid-session (e.g. "MCP server does not exist" or its
  `STATUS.md` reports an error). Check the status: if it needs authentication, call its
  `mcp_auth` tool. If it remains absent from the session's server list, retrying will not
  help — the list is fixed for the life of a session. Tell the user to reconnect the
  server in Cursor Settings and **start a fresh chat** to pick it up, then either continue
  with the available sources (clearly noting the gap in the report and "By the numbers")
  or pause until they re-run.
