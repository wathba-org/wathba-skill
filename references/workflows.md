# Wathba governed MCP and CLI agent-workspace workflow

Use `--json` and normally `--no-input`. Keep credentials outside the agent.

## 1. Discover MCP setup

```sh
wathba mcp --api-url https://api.wathba.info --json
```

The response contains setup for:

- Replit: add a custom remote MCP URL and complete browser OAuth.
- Claude Code: `claude mcp add --transport http wathba
  https://api.wathba.info/mcp`, then authorize with `/mcp`.
- Codex: `codex mcp add wathba --url https://api.wathba.info/mcp`, then
  `codex mcp login wathba --scopes mcp:read`.
- MCP Inspector: connect to the same Streamable HTTP endpoint and complete
  OAuth.
- Any other host: use Streamable HTTP, OAuth authorization code with PKCE, and
  the `mcp:read` scope.

Do not put a Wathba CLI token or project API key in the MCP host configuration.

## 2. Recommend a service and resolve a project

From the repository root, run:

```sh
wathba service recommend --project-dir . --json --no-input
```

The exact outbound `RepositoryProfileV1` is included in the result for audit.
It contains only bounded identifiers, allowlisted evidence, architecture
booleans, and environment-variable names. If there is no project, run the
returned explicit `wathba project create` command with a stable idempotency key,
then repeat recommendation with `--project <projectId>`. Recommendation itself
never enables a service or creates a project.

Through MCP, `recommend_services_for_repository` uses the same profile and
policy. MCP `create_project` is the only mutation and requires separately
approved `projects:create`; never request it when a project already exists.

## 3. Read project and service facts

Call `list_projects`, select the exact project ID, then call
`get_project_setup` and `list_project_services`. For a selected service, call
`get_service_integration_docs`, `get_service_operations`, and
`get_service_troubleshooting` as needed.

Treat returned service, skill, operation, cost, limit, and environment pins as
authoritative. Missing, ambiguous, mismatched, or unknown facts fail closed.

## 4. Resolve the exact integration bundle

```sh
wathba integrate inspect --project-dir . --json --no-input
wathba integrate <capabilityCode> --project-dir . --project <projectId> --environment <environmentId> --json --no-input
```

Inspection does not upload or modify repository content. It recognizes
TypeScript, JavaScript, Python, Java, Go, PHP, .NET, cURL-oriented, and unknown
projects. `integrate` checks the exact project/environment/service/capability
pins, signed artifacts, readiness, recipe, and verification profile, then
installs the exact trusted skill by default. It never patches application code
or calls a runtime operation.

## 5. Implement and verify

Patch the member application with its normal coding tools. Keep Wathba calls in
trusted server-side code. Inspect the pinned runtime projection with `wathba
capability operations`, then validate local JSON without uploading it:

```sh
wathba capability validate <capabilityCode> --operation <operationId> --request request.json --project <projectId> --environment <environmentId> --json --no-input
wathba capability verify <capabilityCode> --mode contract --idempotency-key <stable-key> --project <projectId> --environment <environmentId> --json --no-input
```

Contract verification has no provider effect. Sandbox mode performs one
governed real-sandbox probe and requires `--accept-provider-effect`.

MCP itself can be tested safely by connecting MCP Inspector and listing all
eight tools and three resources. Seven tools are read-only. Test
`create_project` only in an approved no-project sandbox journey with a stable
idempotency key; attempting an unknown or under-scoped mutation must fail.

## 6. Hand off to the member

Report:

- which project, environment, service, skill pin, and operation contract were
  used;
- which local tests passed;
- what the member must do in the portal.

The member creates the production environment, completes production approval,
creates the one-time key, stores it directly in the trusted server secret
store, and deploys. Do not ask the member to reveal the key.

Legacy `.wathba` journal or `READY` files are ignored. The CLI does not track
integration progress or certify production readiness.
