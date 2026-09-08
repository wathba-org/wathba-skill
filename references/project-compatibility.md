# Repository detection and integration modes

Run:

```sh
wathba integrate inspect --project-dir . --json --no-input
wathba service recommend --project-dir . --json --no-input
```

The detector is local and read-only. It does not upload repository content,
install dependencies, modify files, or require Wathba authentication.

| Detected stack | MCP guide mode |
|---|---|
| TypeScript | Pinned Wathba SDK plus HTTP contract |
| JavaScript | Pinned Wathba SDK plus HTTP contract |
| Python | Direct HTTP |
| Java | Direct HTTP |
| Go | Direct HTTP |
| PHP | Direct HTTP |
| .NET | Direct HTTP |
| cURL-oriented | Direct HTTP |
| Other/unknown | Portable direct HTTP |

Detection uses familiar repository markers such as `package.json`,
`pyproject.toml`, `requirements.txt`, `pom.xml`, `build.gradle`, `go.mod`,
`composer.json`, and `*.csproj`. It does not require a specific framework or
package manager.

After recommendation and explicit project/environment selection, run `wathba
integrate <capabilityCode> --project-dir . --project <projectId> --environment
<environmentId> --json --no-input`. It authenticates the exact pinned version 2
bundle and signed skill. Do not reuse an old SDK version, service mapping, or
operation list from a local file.

## API contract compatibility

Every enabled service binding owns an `apiContractVersion`. That value, not the
catalog default and not the CLI or SDK release date, determines the member-app
request and response contract. Repository integration must read and preserve it:

```sh
wathba capability api-contract inspect <serviceCode> \
  --project <projectId> --environment <environmentId> --json --no-input
```

Generated runtime calls send the exact value in `Wathba-Version`. They accept
only a response carrying the same header. A missing or different version is a
compatibility failure, not a reason to retry against the newest contract.

Catalog repins, provider adapter releases, CLI/SDK updates, and production
deployments do not change an existing binding pin. New bindings receive the
then-current default. Changing a sandbox pin requires sandbox fixture proof, a
stable idempotency key, the expected binding owner version, and an explicit
upgrade or rollback command. Production upgrade and rollback are portal-only;
direct the member to the trusted portal.

Idempotent replays retain the contract version and normalized request
fingerprint stored with the original execution. Never reconstruct a replay with
today's default contract.
