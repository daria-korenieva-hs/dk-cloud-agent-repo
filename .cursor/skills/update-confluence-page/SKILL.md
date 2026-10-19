---
name: update-confluence-page
description: Updates an existing Hootsuite Confluence page from a Slack thread or an explicit request. Use when the user asks to update, edit, or rewrite a Confluence page, or says to use update-confluence-page.
---

# Update Confluence page

Edit an existing Confluence page. Do not create a new page unless the user explicitly asks.

## Steps

1. Resolve the page. Prefer a URL or page ID the user (or Slack thread) already gave. Otherwise search with `searchConfluenceUsingCql` (`type = page AND title ~ "…"`). Confirm the match if more than one page looks right.
2. Fetch the current page with `getConfluencePage` (`cloudId`, `pageId`, `contentFormat: "markdown"`) so the edit is based on what is there now.
3. Resolve `cloudId` with `getAccessibleAtlassianResources` if it is not already known. Hootsuite site: `https://hootsuite.atlassian.net`.
4. Draft the replacement body. Keep the user's requested change; do not rewrite unrelated sections.
5. Before writing HTML, call `getContentFormatGuide` with `{ "toolName": "updateConfluencePage" }` and follow that guide. Skip the guide only if you pass `contentFormat: "markdown"`.
6. Call `updateConfluencePage` with `cloudId`, `pageId`, and `body`. Set `versionMessage` to a short reason for the edit.
7. Reply with the page title, a one-line summary of what changed, and the `webUrl`.

## Do not

- Overwrite the page with unrelated content.
- Move the page (`spaceId` / `parentId`) unless the user asked.
- Invent a page ID or title.
