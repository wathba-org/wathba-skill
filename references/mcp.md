# Wathba over MCP only

Use this runbook when the host exposes the Wathba MCP tools but has no shell or
`wathba` CLI (a chat or desktop host, or any remote-MCP client). The CLI
commands in `SKILL.md` then do not apply; its safety rules, notice policy, and
service rules still do. Never ask the member to install the CLI just to read
project facts or integration guidance.

## Connect

The Wathba plugin for Claude Code and Codex adds the hosted server; other hosts
add the remote MCP server `https://api.wathba.info/mcp`. The host opens Wathba
sign-in in the browser (OAuth with PKCE) and the member approves the narrow
`mcp:read` scope. No API key or token ever goes through the chat. If a tool
reports an authentication error, ask the member to reconnect the Wathba server
in the host; never ask for a password, code, or key.

## Tools

The base `mcp:read` grant serves exactly nine tools:

| Tool | Use it to |
| --- | --- |
| `list_notices` | Run the member-wide attention check; page with `cursor`, then `portfolioCursor`. |
| `list_projects` | List the member's projects and environments. |
| `get_project_setup` | Read sandbox, production, portal, and API-key guidance (never credentials). |
| `list_project_services` | Read `configuredServices` and `availableServices` for a project. |
| `get_service_integration_docs` | Read the pinned integration guide for a configured service. |
| `get_service_operations` | Read only that guide's runtime operations (never executes one); skip it after reading the guide. |
| `get_service_troubleshooting` | Read only that guide's troubleshooting and stable error codes; skip it after reading the guide. |
| `recommend_services_for_repository` | Recommend services from a local `RepositoryProfileV1`. |
| `create_project` | Create one sandbox project (needs `projects:create`). |

`create_project` needs the separately approved `projects:create` scope, and
only when no suitable project exists. Without it the tool does not fail: it
returns `outcome: "authorization_required"` with `requiredScopes`. Ask the
member to re-authorize this Wathba connection (reconnect or re-consent in the
host), keeping its current scopes (always `mcp:read`) and adding
`requiredScopes`, so for `create_project`: `mcp:read projects:create`. Then
retry with the same `idempotencyKey`. Never work around it. The API upgrade
tools (`inspect_api_upgrade`, `preview_api_upgrade`, `prepare_api_upgrade`,
`apply_api_upgrade`, `rollback_api_upgrade`) are listed only when the
connection has the `mcp:api-contracts:upgrade` scope; calling one without it
fails with an HTTP 403 `insufficient_scope` challenge naming
`mcp:read mcp:api-contracts:upgrade`, handled the same way: keep the current
scopes and add the named ones. Every scope set must include `mcp:read`;
`mcp:read projects:create mcp:api-contracts:upgrade` is also valid. The hosted
MCP serves no domain tools for now.

## Workflow

1. Call `list_notices` first and page until the cursors are null; follow the
   notice policy in `SKILL.md`.
2. Call `list_projects` and agree the project and environment with the member.
   Use the exact IDs from results; never guess or switch them.
3. If you can read the repository locally, build a bounded
   `RepositoryProfileV1` and call `recommend_services_for_repository`. Never
   send source code, file contents, environment values, credentials, or git
   history. Without repository access, ask the member what the app needs.
4. Call `list_project_services`. For a service the project lacks, give the
   member its portal `nextAction.url`: MCP never enables a service.
5. For a configured service, call `get_service_integration_docs` with the
   `integrationDocs` arguments `list_project_services` returned. Keep
   `projectId`, `environmentId`, `serviceCode`, `capabilityCode`, and
   `contractVersion` exactly; they pin the right service and contract
   version. Set `stack` to the member app's server-side stack (`typescript`,
   `javascript`, `python`, `java`, `go`, `php`, or `dotnet`) and `language` to
   the member's language (`ar` or `en`) when known; the returned `curl` and
   `en` are defaults for when they are unknown. If a stack is not offered for
   that service, call again with `stack: "curl"`. The guide already contains
   the service's operations and troubleshooting: do not also call
   `get_service_operations` or `get_service_troubleshooting` for it, since they
   return subsets of the same guide. Follow only the pinned guidance; never
   invent endpoints, fields, or prices.
6. Edit the member's code only where the host gives you the workspace;
   otherwise hand over the exact code and steps from the pinned guide.
7. Call `list_notices` again after setup and before the final handoff.

## Hand off to the member

These tools read and guide. They never enable a service, create or reveal an
API key, fund the wallet or read its balance (the notice code
`wallet_check_requires_portal` says so), set or raise a project's spending
limit, accept provider terms, create or
promote production, deploy, or run a runtime operation. For each such step,
give the member the portal link from the result or notice, say what it
unblocks, and read fresh notices afterwards. A member who has a shell can do
the same steps with the Wathba CLI described in `SKILL.md`.

## Results

Every result, including errors, is one JSON object in `structuredContent`:
`result` on success, or `error` with `code`, `message`, `retryable`,
`billingEffect`, and `correlationId`, plus the notice fields; `content` is
always empty. The server instructions carry a short Wathba orientation and the
same notice policy.

## Language

Results and this skill are English. Reply in the member's language (Arabic or
English) and keep codes, IDs, tool names, and links verbatim; see
`references/arabic-glossary.md`.
