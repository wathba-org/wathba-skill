---
name: wathba
description: "Install and operate the Wathba (وثبة) CLI for credential-safe member workspace discovery, operator-enabled services, signed capability integration, runtime operation discovery, webhooks, and safe API-key metadata. Use whenever the user mentions Wathba, وثبة, wathba-cli, a Wathba service or capability, or asks in Arabic or English to connect OTP, payments, shipping, Moyasar, Torod, or Authenta through Wathba."
---

# Wathba CLI

Use `wathba` as the agent interface to Wathba. Respond in the user's language;
keep commands, IDs, codes, URLs, and JSON fields in Latin script.

## Core safety rules

1. Use `--json` and normally `--no-input`. Parse the envelope; do not scrape
   human help text.
2. Inspect the typed outcome as well as the exit code.
3. Never ask for, receive, print, store, or paste a project API key, provider
   credential, password, cookie, signing secret, raw card data, protected
   document, funding detail, or raw provider payload.
4. Never invent a provider route or operation. Use `wathba manifest --json`,
   `wathba api operations --json`, and the verified signed service manifest.
5. Service onboarding is operator-owned for the MVP. Do not search for or
   invent member `setup`, `open`, `reconcile`, `activate`, or `deactivate`
   commands.
6. Treat `wathba service list --json --no-input` as the only current service
   inventory. Named services in these bundled references are integration
   examples, not availability claims. If a service is absent, do not advertise,
   inspect, install, or integrate it; treat a direct not-found response the same
   way.

## Installation

Wathba supports native installer scripts and the official zero-dependency npm
package. Do not use Homebrew or customer GitHub Release assets directly:

```sh
# macOS / Linux
curl -fsSL https://install.wathba.info/install.sh | bash

# Windows PowerShell
irm https://install.wathba.info/install.ps1 | iex
```

```sh
# npm on macOS, Linux, or Windows (Node.js 18.18+)
npm install --global @wathba-cli/cli
```

The native installer writes the binary to `$HOME/.wathba/bin` and installs this
skill plus the signed `wathba-integration` bootstrap skill. npm installs the
same authenticated native binary inside `@wathba-cli/cli`, exposes `wathba`,
and intentionally does not write agent skills outside the package. For npm,
install them explicitly when needed:

```sh
wathba skill agent install --json --no-input
wathba skill bootstrap install --json --no-input
```

Pin a native channel or version with `WATHBA_CHANNEL=beta` or
`WATHBA_VERSION=v1.4.0`; use `@beta` or an exact npm version for npm. Update an
npm-managed installation with `npm install --global @wathba-cli/cli@latest`,
not `wathba self-update`. Both installation paths verify signed artifacts;
never bypass verification. Then run:

```sh
export PATH="$HOME/.wathba/bin:$PATH"
wathba version --json
wathba doctor --json
```

## Authenticate and select context

```sh
wathba login --no-input --json
# After the member approves the code in Wathba:
wathba auth complete --no-input --json
wathba auth status --json
wathba workspace show --json --no-input
wathba project select <projectId> --environment <environmentId> --json
```

The backend assigns the fixed `member_workspace.v2` profile; login never accepts
raw scopes or a selectable profile. Tokens stay in the OS keychain. Workspace
and integration commands reject `--token` and `WATHBA_TOKEN`. In an agent run,
use `wathba login --no-input --json`, present its safe approval URL and code to
the member, then use `wathba auth complete --no-input --json` after approval.

## Service enablement

First read the live service inventory. Wathba backoffice operators complete
provider onboarding and enable services returned there:

- Authenta/Authentica and Torod: enable once for the member; all member projects
  can use them.
- Moyasar-backed payments: enable separately for each project.

The member/agent commands are read-only:

```sh
wathba service list --project <projectId> --json --no-input
wathba service status <serviceCode> --project <projectId> --environment <environmentId> --json --no-input
wathba service wait <serviceCode> --until enabled --project <projectId> --environment <environmentId> --json --no-input
wathba service skill <serviceCode> --project <projectId> --json --no-input
```

If the live inventory contains the service but it is not enabled, report the
exact operator action from the output. Do not claim the member can fix it from
the CLI. Status and wait never mutate provider state.

## Integrate a capability

```sh
wathba integrate <capabilityCode> \
  --project <projectId> --environment <environmentId> \
  --project-dir . --json --no-input
wathba integrate resume <capabilityCode> --project-dir . --json --no-input
wathba integrate status <capabilityCode> --project-dir . --json --no-input
wathba integrate verify <capabilityCode> --project-dir . --json --no-input
```

`integrate` first confirms the operator-enabled service projection. A disabled
service causes no skill installation or project patch. Once enabled, the CLI
verifies catalog pins, the signed manifest, signed skill, and signed patch
recipe. It retains only member-owned budget and webhook actions, applies the
patch, and verifies the result.

The signed manifest's full `operations[]` set is authoritative. New Torod,
Moyasar, or Authenta operations do not require a hardcoded CLI update. Reject a
signature, identity, digest, schema, or classification mismatch; never replace
the signed bundle with a local one.

Integration state is a minimal journal in the user config directory plus
`.wathba/integration.lock` in the project. On a conflict, run `integrate status`
and continue from current state instead of retrying blindly.

## Runtime and API keys

Runtime operations execute from the member application's trusted server with a
separate project/environment API key. An authorized human creates and reveals
that key once at `/app/projects/<projectId>/keys` and configures it directly on
the server outside the agent's view.

The fixed workspace profile includes `keys:read` and intentionally excludes
`keys:manage`. An agent may use only `wathba key list --project <projectId>
--json` for safe metadata. Key creation, rotation, revocation, suspension,
reactivation, and compromise handling are protected portal actions. A workspace
agent must not invoke them or seek a broader token.

The runtime path is member app → Wathba → server-side provider credential →
provider → normalized response. The agent never calls the provider directly.

## Torod

Only when the live inventory contains `shipping.torod`, Torod is
operator-enabled once per member. The CLI has no Torod login, plugin,
address, readiness-refresh, wallet-facts, funding-link, or wallet-funding flow.
Funding remains between the member and Torod through Torod's operation.

All Wathba-supported Torod runtime operations come from the signed service
manifest. If a runtime result is pending or ambiguous, poll the same Wathba
execution; do not issue a duplicate shipment as a probe.

## Webhooks

Webhooks are retained. The member configures a Wathba delivery endpoint and
subscriptions. Wathba registers supported provider webhooks using server-held
credentials, verifies inbound callbacks, deduplicates and normalizes them, and
delivers signed member events. Never request a provider webhook secret.

## Troubleshooting

- Authentication required: run `wathba login --no-input --json`, ask the member
  to approve it, then run `wathba auth complete --no-input --json`.
- Disabled service: report whether the Wathba operator must enable it for the
  member or selected project; optionally use `service wait --until enabled`.
- Integration conflict/drift: run `integrate status`, inspect the project, and
  resume only from the recorded context.
- Protocol/signature failure: stop. Do not bypass verification.
- Unknown command or flag: inspect `wathba manifest --json` or command help.

## References

- `references/commands.md` — command and flag reference.
- `references/workflows.md` — end-to-end operator-enablement, integration,
  runtime, and webhook workflows.
- `references/arabic-glossary.md` — Arabic intent mapping and response style.
