# Summarizing Contributions Skill 

### About

A personal Cursor Agent Skill that turns your work over a time period into a themed,
impact-focused markdown report, pulled from your **merged GitHub PRs**, **resolved Jira
tickets**, and **created Confluence pages**.

It's built for self-review / brag-doc style summaries: contributions are grouped by impact
area (not by source), and every claim links back to a PR, ticket, or page.

---

## Files in this skill

| File           | Purpose                                                                       |
| -------------- | ----------------------------------------------------------------------------- |
| `SKILL.md`     | What Cursor loads: the trigger description + the 6-step workflow.             |
| `reference.md` | The exact query recipes, identity resolution, date math, and output template. |

---

## Prerequisites

The skill calls existing MCP servers; it does **not** store any credentials of its own.
You need two integrations connected in Cursor:

1. **GitHub MCP** — provides `get_me`, `search_pull_requests`, `pull_request_read`.
2. **Atlassian MCP** (Jira + Confluence) — provides `jira_search`, `jira_get_issue`,
   `jira_get_user_profile`, `confluence_search`, `confluence_get_page`.

If only one is connected, the skill still runs and clearly notes the missing source in the
report and the "By the numbers" footer.

---

## Setting up the MCPs

MCP servers are configured in `~/.cursor/mcp.json` and managed under
**Cursor Settings → Tools & MCPs**. Below is the setup that this skill was
validated against.

### GitHub (HTTP + OAuth)

In `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "<url from awesome zendesk ai>",
      "headers": {}
    }
  }
}
```

This server authenticates via OAuth. The first time you use it, the agent (or the MCP
panel in Settings) will prompt you to authenticate — in-agent this happens by calling the
server's `mcp_auth` tool, which opens the sign-in flow. Once authenticated it shows green
in Settings → MCP.

> The skill only needs read access to your PRs.

### Atlassian — Jira + Confluence (Docker + env file)

In `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "atlassian-mcp": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--env-file", "<path-to-.claude/mcp-atlassian.env>",
        "ghcr.io/sooperset/mcp-atlassian:latest"
      ]
    }
  }
}
```

This uses the [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian)
image. Requirements:

- **Docker** installed and running.
- An **env file** (here `~/.claude/mcp-atlassian.env`) holding your Cloud URLs and API
  tokens. Create API tokens at <https://id.atlassian.com/manage-profile/security/api-tokens>.
  Template:

```bash
# Jira
JIRA_URL=https://your-domain.atlassian.net
JIRA_USERNAME=you@example.com
JIRA_API_TOKEN=your_jira_api_token

# Confluence
CONFLUENCE_URL=https://your-domain.atlassian.net/wiki
CONFLUENCE_USERNAME=you@example.com
CONFLUENCE_API_TOKEN=your_confluence_api_token
```

> Keep the env file out of version control. After editing `mcp.json` or the env file,
> toggle the server off/on in Settings → MCP (or restart Cursor) so it reloads.

### Verifying

In Settings → MCP, both servers should show **connected (green)** with their tools listed.
You can sanity-check from a chat: "call `get_me`" (GitHub) and "run a `jira_search` for
`assignee = currentUser()`".

---

## Using the skill

Invoke it by name in any chat, with a period:

```
Use summarizing-contributions to summarize my contributions for Q1 2026.
```

Accepted period formats (Step 1 resolves these to concrete dates and confirms before
querying):

- Natural language: `last quarter`, `Q1 2026`, `May 2026`, `last 30 days`
- Explicit range: `2026-01-01 to 2026-03-31`

What it does:

1. Resolves the date range and states it back to you.
2. Resolves identity — GitHub `login` via `get_me`; Jira/Confluence use `currentUser()`.
3. Gathers merged PRs, resolved tickets, and created pages for the window.
4. Reads descriptions to understand impact.
5. Clusters everything into 3–6 themes.
6. Writes `~/contributions/contributions-<period>.md` and prints the path.

### Output

A markdown file under `~/contributions/`, e.g.
`~/contributions/contributions-2026-01-01_2026-03-31.md`, structured as:

- **Summary** — a few sentences of overall impact
- **3–6 theme sections** — narrative + linked PRs/tickets/pages as evidence
- **By the numbers** — counts per source

---

Guidelines when editing:

- Keep `SKILL.md` concise (well under 500 lines) — push detailed/fragile bits into
  `reference.md`. Section names cited from `SKILL.md` must match headings in `reference.md`.
- After editing, start a fresh chat so Cursor reloads the skill, then do a small dry-run
  (e.g. `last 30 days`) and spot-check that a couple of links match their claims.
- To verify tool names/parameters before relying on them, inspect the MCP tool schemas
  under `~/.cursor/projects/<workspace>/mcps/<server>/tools/`.

---

## Troubleshooting

- **"MCP server does not exist" mid-run** — the server isn't loaded in the current chat
  session. Reconnect it in Settings → MCP, then **start a fresh chat** (the server list is
  fixed for the life of a session; retrying in the same chat won't pick it up).
- **Auth error / server needs login** — the agent should call that server's `mcp_auth`
  tool; otherwise authenticate it from Settings → MCP.
- **Atlassian server errored** — confirm Docker is running and the env file path/tokens in
  `mcp.json` are valid. API tokens expire/rotate; regenerate if needed.
- **Empty results** — confirm the period actually contains merged PRs / resolved tickets /
  created pages, and that your `currentUser()` (Atlassian) and GitHub `login` map to the
  accounts where that work lives.
- **Missing a source in the report** — that source was unavailable or returned nothing;
  the "By the numbers" footer will say so. Reconnect it and re-run to fold it in.
