---
name: using-atlassian
description: Queries Jira and Confluence through the read-only Atlassian MCP server. Use for Jira issues, JQL, projects, fields, Confluence pages, spaces, content search, and Atlassian links.
mcpServers:
  atlassian:
    command: npx
    args: ["-y", "@andreimaxim/atlassian-mcp@0.1.0"]
    env:
      ATLASSIAN_SITE_URL: "${ATLASSIAN_SITE_URL}"
      ATLASSIAN_EMAIL: "${ATLASSIAN_EMAIL}"
      ATLASSIAN_API_TOKEN: "${ATLASSIAN_API_TOKEN}"
    includeTools: ["atlassian_get", "jira_search"]
---

# Using Atlassian

Use the Atlassian MCP tools for bounded, read-only Jira and Confluence work.

## Choose the tool

- Use `jira_search` for JQL. It is the only read-only POST endpoint and returns Jira's raw search response.
- Use `atlassian_get` for every other Jira or Confluence read.
- Writes are not supported. Never attempt mutations through `atlassian_get` or `jira_search`.

## Search with JQL

Keep searches bounded because Jira rejects unbounded JQL in some installations. Request only fields needed for the answer.

Examples:

- Most recently updated ticket assigned to the current user:
  - JQL: `assignee = currentUser() AND updated >= -365d ORDER BY updated DESC`
  - `maxResults`: `1`
  - `fields`: `["summary", "status", "assignee", "updated"]`
- Current user's unresolved work:
  - JQL: `assignee = currentUser() AND resolution IS EMPTY AND updated >= -365d ORDER BY priority DESC, updated DESC`
- Recently changed project work:
  - JQL: `project = FWS AND updated >= -30d ORDER BY updated DESC`

When asked for “the last ticket I worked on,” explain that Jira search can identify the most recently updated ticket assigned to the user, but does not prove who made the latest update. Use changelog or worklog data if the user specifically means their own recorded activity.

Pass `nextPageToken` from a response into the next `jira_search` call when another page is needed.

## Common GET requests

Call `atlassian_get` with these site-relative paths:

- Issue: `/rest/api/3/issue/{issueKey}` with query `fields: "summary,status,assignee,description,updated"`
- Issue changelog: `/rest/api/3/issue/{issueKey}/changelog`
- Available transitions: `/rest/api/3/issue/{issueKey}/transitions`
- All fields: `/rest/api/3/field`
- Fields editable on an issue: `/rest/api/3/issue/{issueKey}/editmeta`
- Project: `/rest/api/3/project/{projectKey}`
- Project issue types: `/rest/api/3/issuetype/project` with `projectId` and `level: 0`
- Assignable users: `/rest/api/3/user/assignable/search` with `issueKey`, `query`, and a bounded `maxResults`
- Current user: `/rest/api/3/myself`

Use the response's pagination token, cursor, `startAt`, or `_links.next` according to that endpoint. Make one GET per tool call.

## Common Confluence requests

Call `atlassian_get` with site-relative paths beginning with `/wiki/`:

- Page by ID: `/wiki/api/v2/pages/{pageId}`
- Pages in a space: `/wiki/api/v2/spaces/{spaceId}/pages`
- Spaces: `/wiki/api/v2/spaces`
- Page children: `/wiki/api/v2/pages/{pageId}/children`
- Page attachments: `/wiki/api/v2/pages/{pageId}/attachments`
- CQL content search: `/wiki/rest/api/content/search` with a bounded `cql` query and `limit`

When page content is needed, pass a supported `body-format` query value such as `storage` or `atlas_doc_format`. Follow `_links.next` exactly for pagination.

If asked to create, edit, assign, transition, or delete Atlassian data, explain that the MCP server is read-only and cannot perform the change.
