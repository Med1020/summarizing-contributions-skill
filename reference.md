# Reference: summarizing-contributions

## Identity resolution


| Source     | How to get "me"                                              |
| ---------- | ------------------------------------------------------------ |
| GitHub     | Call `get_me` (no args). Use the `login` field as `<login>`. |
| Jira       | Use `currentUser()` directly in JQL — no lookup needed.      |
| Confluence | Use `currentUser()` directly in CQL — no lookup needed.      |


To display the user's name in the report, optionally call `jira_get_user_profile`
with their email/username and read `displayName`.

## Date-range math

Resolve the period to concrete `start` and `end` dates (inclusive), using today's date
for relative terms:

- `Q1` = Jan 1–Mar 31, `Q2` = Apr 1–Jun 30, `Q3` = Jul 1–Sep 30, `Q4` = Oct 1–Dec 31
(of the stated or current year).
- `last quarter` = the most recently completed calendar quarter.
- `<Month> <Year>` (e.g. `May 2026`) = first to last day of that month.
- `last N days` = (today − N days) to today.
- Explicit `YYYY-MM-DD` start/end = use as given.

Format dates as `YYYY-MM-DD` for all three query languages.

## Query recipes

### GitHub — merged PRs I authored

Tool: `search_pull_requests`. Set `perPage: 100` and increment `page` until a page
returns fewer than 100 results.

```
query: author:<login> is:merged merged:<start>..<end>
sort: created
order: desc
perPage: 100
page: 1   (then 2, 3, ... until exhausted)
```

Keep per result: title, number, repository (owner/repo), merged date, html URL.
Enrich with `pull_request_read` (the PR body) in Step 4.

### Jira — tickets assigned to me, resolved in period

Tool: `jira_search`. `limit` max is 50 — page with `start_at` (0, 50, 100, ...) or
`page_token` until results are exhausted.

```
jql: assignee = currentUser() AND statusCategory = Done AND resolved >= "<start>" AND resolved <= "<end>" ORDER BY resolved DESC
fields: summary,status,resolutiondate,issuetype,priority,labels,description
limit: 50
start_at: 0   (then 50, 100, ... until exhausted)
```

Keep per result: issue key, summary, issue type, resolution date. Build the URL as
`<jira-base-url>/browse/<KEY>`. Enrich with `jira_get_issue` (the description) in Step 4.

### Confluence — pages I created in period

Tool: `confluence_search`. `limit` max is 50 — page until exhausted.

```
query: creator = currentUser() AND type = page AND created >= "<start>" AND created <= "<end>" ORDER BY created DESC
limit: 50
```

Keep per result: page title, page id, created date, and the page URL/`_links`. Enrich
with `confluence_get_page` (the page summary) in Step 4.

## Output template

```markdown
# Contributions — <Period> (<start> to <end>)

## Summary
<2-4 sentence high-level impact overview>

## <Theme 1, e.g. Reliability & Performance>
<short narrative of what was done and why it mattered>
- <PR / ticket / page title> — <one-line impact> ([link](<url>))

## <Theme 2 ...>
<narrative>
- <item> — <impact> ([link](<url>))

## By the numbers
- <N> PRs merged · <N> Jira tickets resolved · <N> Confluence pages created
```

Formatting rules:

- 3-6 themes. Drop a theme if it has no real items.
- Every bullet links to a real item. No fabricated contributions.
- Order themes by impact/volume, most significant first.

