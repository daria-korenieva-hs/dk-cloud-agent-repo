---
name: create-jira-issue
description: Creates a Hootsuite Jira issue from a Slack thread or an explicit request. Use when the user asks to file, create, or open a Jira ticket, or says to use create-jira-issue.
---

# Create Jira issue

Create one Jira issue. Ask before creating if the project, type, or summary is unclear.

## Steps

1. Confirm `projectKey`, `issueTypeName` (Task, Bug, Story, …), and a one-line `summary`. Infer from the Slack thread when those are obvious; otherwise ask.
2. Search for duplicates first (`searchJiraIssuesUsingJql` on the summary / key terms in that project). If a close match exists, link it and stop unless the user still wants a new issue.
3. Resolve `cloudId` with `getAccessibleAtlassianResources` if needed. Hootsuite site: `https://hootsuite.atlassian.net`.
4. Write a short markdown description from the thread or request: context, asked outcome, links. Do not invent acceptance criteria or owners.
5. Call `createJiraIssue` with `cloudId`, `projectKey`, `issueTypeName`, `summary`, and `description` (`contentFormat: "markdown"`). Put labels, priority, components, or custom fields in `additional_fields` only.
6. Reply with the issue key, summary, and browse URL (`https://hootsuite.atlassian.net/browse/KEY-123`).

## Do not

- Create a ticket when a duplicate already covers the request.
- Guess a project key or assignee account ID.
- File in a project the user did not name or imply.
