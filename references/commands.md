# Wathba CLI command reference

Use `--json` for deterministic envelopes and `--no-input` for agent runs.
Global context flags include `--project`, `--environment`, `--api-url`,
`--config`, `--timeout`, and `--idempotency-key`. Flag values override
environment variables, which override non-secret config.

## Discovery and diagnostics

- `wathba version`
- `wathba doctor [--check <name>]`
- `wathba manifest`
- `wathba schema list`
- `wathba schema get <operationId>`
- `wathba api operations`
- `wathba api call <operationId> --input <file>`
- `wathba completions bash|zsh|fish|powershell`

Raw calls accept only operations classified as agent-safe. Interactive-only,
operator, provider-onboarding, and secret-bearing operations fail locally.

## Authentication

- `wathba login [--wait]`
- `wathba auth status`
- `wathba auth complete --no-input --json`
- `wathba auth refresh`
- `wathba auth logout`
- `wathba auth sessions`
- `wathba auth session revoke <sessionId>`

The backend assigns `member_workspace.v2`. Login accepts neither raw scopes nor
a selectable profile. Tokens stay in the OS keychain.

## Workspace and projects

- `wathba workspace show`
- `wathba project create --name <name>`
- `wathba project list`
- `wathba project get <projectId>`
- `wathba project select <projectId> [--environment <environmentId>]`
- `wathba capability catalog`
- `wathba capability list --project <projectId> [--environment <environmentId>]`
- `wathba capability status <capabilityCode> --project <projectId> [--environment <environmentId>]`
- `wathba capability skill <capabilityCode>`
- `wathba capability verify <capabilityCode>`

## Services

The service surface is read-only:

- `wathba service list --project <projectId>`
- `wathba service status <serviceCode> --project <projectId> --environment <environmentId>`
- `wathba service wait <serviceCode> --until enabled --project <projectId> --environment <environmentId>`
- `wathba service skill <serviceCode> --project <projectId>`

Authenta/Authentica and Torod are enabled once per member by a Wathba operator.
Moyasar is enabled per project. There are no CLI setup, browser-open,
reconcile, activation, deactivation, provider-readiness, or funding commands.

## Integration

- `wathba integrate <capabilityCode> --project-dir <dir>`
- `wathba integrate resume <capabilityCode> --project-dir <dir>`
- `wathba integrate status <capabilityCode> --project-dir <dir>`
- `wathba integrate verify <capabilityCode> --project-dir <dir>`

Start and resume require the keychain workspace session. Integration stops
without changing the project when operator enablement is missing. Once enabled,
it installs only verified signed artifacts and discovers the complete runtime
operation set from the signed manifest.

Upgrade, rollback, removal, provider onboarding, wallet facts, and production
promotion are not part of the MVP command surface.

## Skills

- `wathba skill resolve <skillId> --version <version> --digest sha256:<hex>`
- `wathba skill install <skillId> --version <version> --digest sha256:<hex> [--target-dir <dir>]`
- `wathba skill bootstrap install [--target-dir <dir>]`
- `wathba agent install [--target-dir <dir>]`

Artifact signatures, digests, publisher identities, and revocation state are
mandatory. Never bypass trust checks.

## API-key metadata

- `wathba key list`

The key secret is created/revealed once to the authorized human in the portal
The workspace profile includes `keys:read` and intentionally excludes
`keys:manage`. Key creation, rotation, revocation, suspension, reactivation, and
compromise handling are protected portal actions. A workspace agent must not
invoke them.

## Updates

- `wathba update check`
- `wathba self-update [--channel stable|beta] [--verify-only] [--rollback]`

## Exit codes

| Code | Meaning |
|---:|---|
| 0 | Command completed; inspect the typed outcome |
| 2 | Invalid input or missing context |
| 3 | Authentication/authorization required |
| 4 | Remote or transport failure |
| 5 | Verification/protocol incompatibility |
| 6 | Local install/filesystem failure |
| 8 | Conflict or replay mismatch; re-read status |
