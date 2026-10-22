---
name: split-jira-story
description: Splits an existing Hootsuite Jira story into logical subtasks after researching the ticket, related work, and company knowledge. Use when the user asks to split, break down, or decompose a story into subtasks, or says to use split-jira-story.
---

# Split Jira story

Split one existing Jira story into the fewest logical subtasks. Prefer merging items that share the same goal and the same named repo. From Slack: `@Cursor use split-jira-story …` (not a slash command).

## Steps

1. Require a story key or browse URL. If the key is missing, stop and ask.
2. Resolve `cloudId` with `getAccessibleAtlassianResources` if needed. Hootsuite site: `https://hootsuite.atlassian.net`. If the Atlassian MCP returns `needsAuth`, point the user at [cursor.com/agents](https://cursor.com/agents).
3. Fetch the issue with `getJiraIssue`. Pass `fields` including `"*all"` (or explicitly `summary`, `description`, `status`, `issuetype`, `comment`, `subtasks`, `issuelinks`, `parent`). Comments need `"comment"`. Do not invent keys, AC, owners, or repos.
4. Gate on issue type. Refuse Epics and existing subtasks. If the type is Story, proceed. If it is Task, Bug, or another type that can have children, say the type and ask whether to split into children anyway. Do not treat an unknown type as a Story.
5. If a parent or epic key exists, fetch that issue too with the same `fields` for context only. Do not use the epic key in the children JQL. Search children of the issue being split (the Step 1 key) with `searchJiraIssuesUsingJql` (`parent = <issue being split>`). Also search linked or prior tickets and work completed before this story.
6. Invoke `search-company-knowledge` for the company knowledge-base pass: systems named on the story, prior designs, and how similar work was done.
7. Compare what the ticket and prior work already record versus what this story still needs. Treat unfinished tasks on **previous** tickets as done for planning this split: do not wait for them, do not recreate them, and do not stall implementation children because a predecessor is still in progress. Record those unfinished items under **Depends on**. If *this* issue is Done or Cancelled, or its own existing subtasks already cover remaining work, report that and stop. Do not invent subtasks. Do not split a closed issue unless the user insists.
8. Use a research subtask **only** when the work is completely unpredictable (for example a redesign with no prior pattern). A few open questions, missing AC, or an approach that can be inferred from prior tickets or the KB is not enough. Invoke `define-task-ac` (Mode B) only when the user also asked to refine AC. Do not edit the parent story on a split-only run.
9. Decide the split. Prefer fewer tickets. Merge items that share the same goal and the same named repo:
   - Split per repo only when the ticket, parent, or KB names a repo, GitHub URL, or service.
   - If multi-codebase work is implied but unnamed, ask. Do not invent repo names. Do not silently merge implied multi-repo work.
   - Otherwise group by logical workstream. Do not auto-split backend / frontend / tests / docs unless each has a distinct goal.
   - Put a few open questions in an **Open questions** section on the relevant implementation subtask. If there are many, add one dedicated open-questions subtask instead of a research spike.
10. Draft the proposal in chat. Always propose first. Each subtask must include a description of the work, the goal, and a paragraph defining when it is considered done. Add **Depends on** when earlier work (including unfinished tasks on previous tickets) must land first. Add **Open questions** when the item is still implementable but not fully settled. Skip any item an existing subtask on *this* story already covers.
11. Ask before creating. Skip the ask only for an explicit create, file, or go-ahead, or approval of a shown proposal. Invoking this skill or saying “split KEY” is not create-approval.
12. Before writing, call `getJiraProjectIssueTypesMetadata` with `cloudId` and `projectIdOrKey` from the parent story. Use only that project's subtask or child type (`Sub-task` / `Subtask` / equivalent that is a subtask). If none exists, stop and say so. Do not guess. Do not create a sibling Story or Task.
13. Take `projectKey` from the parent story. For each approved item, search duplicates with `searchJiraIssuesUsingJql` on the summary and key terms under `parent = KEY`. On a duplicate, skip that item and continue the batch.
14. Call `createJiraIssue` with `cloudId`, `projectKey`, `issueTypeName` from the metadata lookup, `parent` set to the story key, `summary`, and `description` (`contentFormat: "markdown"`). Write the description as markdown with the fields from step 10 (Description, Goal, Done when, plus Depends on and Open questions when they apply). Do not copy estimate, sprint, or parent-only custom fields. Copy labels, priority, or components only if the user asked or the project requires them. Never invent assignee or project.
15. Reply with the story browse URL and each new key and URL (`https://hootsuite.atlassian.net/browse/KEY-123`).

## Subtask shape

```text
Description: Add the remaining rate-limit headers on the public API already named in the story. Do not change auth.
Goal: Callers can see remaining quota on every successful response.
Depends on: API-2101 (header contract on the gateway) is still in progress on the previous ticket; treat that work as done for this split and do not recreate it here.
Open questions: Confirm whether 429 responses should include the same remaining-quota headers as 2xx.
Done when: Successful public API responses include the documented remaining-quota headers on the surfaces the story already names. Existing auth behavior is unchanged, including rejected and unauthenticated requests. The story's AC for this header contract pass, and no new required headers appear on endpoints the story does not mention.
```

## Do not

- Maximize ticket count, open a research spike when the work is still implementable, wait on unfinished previous-ticket work before proposing children, or split by layer without a separate goal.
- Recreate unfinished tasks from previous tickets, or invent Depends on / Open questions that the ticket, parent, or KB does not support.
- Invent issue keys, acceptance criteria, owners, repositories, assignees, or project keys.
- Treat skill invoke or “split KEY” as create-approval, or create before a shown proposal is approved.
- Run `create-jira-issue` as a full workflow, interview for Task/Bug/Story, or stop the whole batch on one duplicate.
- Invoke `define-task-ac` or edit the parent story unless the user also asked to refine AC.
- Copy estimate, sprint, or parent-only custom fields onto children.
- Use `spec-to-backlog` (Confluence spec → Epic + many tickets) or `capture-tasks-from-meeting-notes`.
- Run `do-tech-plan` unless the user separately asks for a code plan.
