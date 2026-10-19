# dk-cloud-agent-repo

Personal default repo for Cursor Cloud Agents. A Cloud Agent clones this repo into a fresh VM; it cannot see `~/.cursor` on the laptop.

## Layout

```text
.cursor/
  rules/     # always-on conventions
  skills/    # workflow skills (each folder has SKILL.md)
AGENTS.md    # this file
```

## How to work

- Prefer skills in `.cursor/skills/` for repeatable workflows. From Slack, invoke them in natural language (`@Cursor use update-confluence-page …`).
- Use the Atlassian HTTP MCP for Jira and Confluence. Desktop MCP login does not count — Cloud Agents must be authenticated at [cursor.com/agents](https://cursor.com/agents).
- Hootsuite Atlassian cloud: `https://hootsuite.atlassian.net` (resolve `cloudId` with `getAccessibleAtlassianResources` if needed).
- Fetch the current Jira issue or Confluence page before editing. Do not invent issue keys, page IDs, or body content.
- Ask before creating a Jira issue or a new Confluence page unless the user clearly asked for that write.
- Never commit secrets, tokens, or `.env` files.

## Non-goals

This repo is for personal / non-code tasks (docs, tickets, Slack follow-up). Do not copy Hootsuite marketplace plugin caches here. Product code lives in the service repos.
