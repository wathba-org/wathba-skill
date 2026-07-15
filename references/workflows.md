# Wathba CLI — End-to-End Workflows

Always run commands with `--json` (and `--no-input` unless you specifically want a browser handoff). After every command: check the exit code, then parse the envelope, then branch on `data.outcome`. Exit 0 alone proves nothing.

## Outcome state machine

```
SETUP_NOT_REQUIRED ──────────────► activate ─► ACTIVE
setup ─► PENDING ─► READY_TO_ACTIVATE ─► activate ─► ACTIVE
   │        │
   │        ├─► ACTION_REQUIRED  (member setup page; exact `service open` next command)
   │        └─► BLOCKED          (precondition failed; hint says what)
deactivate ─► DEACTIVATION_PENDING ─► wait --until removed ─► REMOVED
```

Also check `data.canActivate` before calling `activate`.

## 0. Fresh machine → working CLI

```sh
curl -fsSL https://install.wathba.info/install.sh | bash
export PATH="$HOME/.wathba/bin:$PATH"       # persist in the user's shell profile
wathba version --json
wathba doctor --json                         # all checks should pass
```
Scripted install: `WATHBA_INSTALL_OUTPUT=json` and read the final NDJSON event's `outcome`.

## 1. Onboarding (auth + project)

```sh
wathba login --device --wait --json     # prints verification URL + user code; --wait blocks until approved
wathba auth status --json               # confirm session
wathba project create --name "my-app" --json
wathba project select <projectId> --json
```
- `project select` re-reads the project and persists its backend default
  environment. It replaces a stale environment already present in local config.
- Tokens live in the OS keychain only. `--no-input` + login: the device flow still needs the user to visit the URL — surface the URL and code to the user (in their language), then `wathba auth complete` / re-check `auth status`.
- Exit 3 anywhere later → session expired → `auth refresh`, else `login --device` again.

### Workspace-agent onboarding

When the agent needs to inspect the member's projects and capabilities and
perform governed member operations, request the exact workspace profile:

```sh
wathba login --device --access workspace --wait --json
wathba auth status --json
wathba workspace show --json --no-input
wathba capability catalog --json --no-input
wathba capability list --project <projectId> --environment <environmentId> --json --no-input
```

The member approves one named, versioned profile in the Wathba portal. The CLI
stores the resulting session only in the OS keychain and rejects `--token` and
`WATHBA_TOKEN` for workspace commands. `AUTHORIZATION_REQUIRED` means stop and
ask for profile approval; never construct a broader scope list. A backend `403`
is authoritative. Project runtime execution is a different trust domain and
requires a separate project API key.

## 2. Service lifecycle (platform-side on/off)

```sh
wathba service list --json                       # find serviceCode; exact workspace keychain session
wathba service setup <svc> --json
```
Branch on outcome:
- `SETUP_NOT_REQUIRED` → `wathba service activate <svc> --json` directly.
- `PENDING` → `wathba service wait <svc> --until ready --json` (blocks; respects `--timeout`), or poll `service status`.
- `ACTION_REQUIRED` → `wathba service open <svc> --json --no-input` gives the exact channel-validated Wathba setup URL with selected `environmentId`; hand it to the user, then re-check `status`.
- `READY_TO_ACTIVATE` / `canActivate: true` → `wathba service activate <svc> --json`.
Finish: `wathba service status <svc> --json` must show `ACTIVE` before you tell the user it's done.

Deactivation: `service deactivate` → outcome `DEACTIVATION_PENDING` → `service wait <svc> --until removed --json` → `REMOVED`. This is also the canonical capability-removal flow for workspace agents; direct pin removal is legacy browser-fresh and interactive-only.

### Torod setup and safe recovery

`logistics.shipping` maps to `shipping.torod`. Current setup authority is
`shipping.torod.v2` version 2 with exactly these four gates:

- `torod_account_connected`
- `torod_reference_data_fresh`
- `torod_pickup_address_active`
- `torod_wallet_funded`

Historical `shipping.torod.v1` setup, activation, and reconciliation mutations
remain status-only/blocked. An already-active exact v1 runtime may continue
only under its immutable pin and the original five live gates (the four above
plus `torod_webhook_registered`); deactivation remains allowed.
A current v2 projection must contain exactly those four facts;
missing, extra, or webhook-substituted readiness is a protocol failure.

In the `wathba.output.v1` success envelope, use only these stable status paths:

- `data.activationPolicy.setupFlowCode`
- `data.activationPolicy.setupFlowVersion`
- `data.setup.requirements[]`, with each gate's `code`, `status`, `owner`, and
  `blocking`

Every status evaluation pairs the strict Projects binding with the separate
Provider Connections setup projection. Their project, environment, service,
setup reference, and lifecycle pin must agree; neither response authorizes
readiness alone.

For current Torod, the authority paths must be `shipping.torod.v2` and `2`, and
the requirement-code set must be exactly the four gates above. If the authority
paths instead report `shipping.torod.v1` and `1`, keep the command status-only:
do not run `setup`, `activate`, or `reconcile`. If the exact v1 binding is
already active, runtime may continue only while its immutable pin and live
readiness hold. Deactivation remains a separate allowed member-requested
cleanup lifecycle. No v1-to-v2 migration is currently published, so report the
blocked setup/activation/reconciliation mutation and do not invent a
replacement command.

New Torod registration (install-plugin) and existing-account login are done in
the Wathba setup page. The member—not the agent—enters any Torod password,
address/contact data, or wallet funding details. Wathba handles the provider
outcomes exactly:

- install-plugin `status: true`, code `200`: seal the returned plugin
  credentials server-side before completing `torod_account_connected`; Torod
  emails the member's Torod email and generated password.
- install-plugin code `422` with `email: "E-Mail already exist"`: offer
  existing-account login and do not retry registration or create duplicate
  data.
- login-plugin `status: true`, code `200`: seal the returned credential bundle
  server-side before completing the account gate.
- login-plugin code `406` with `"The email or password isn't correct"`: keep
  the gate incomplete and show one generic invalid-credentials portal error.

A successful login can create a fresh Torod app/store association. Treat login
as a non-idempotent provider mutation: reuse a valid current connection when
possible and reconcile an ambiguous result instead of blindly submitting it
again.

Submitting `webhook_url` during install or login is request evidence only. It
is neither a v2 readiness gate nor proof of authenticated callback trust.
Authenticated Merchant polling is authoritative; an unverifiable callback may
only prompt polling and must not mutate readiness, shipment, or money state.

Never request, repeat, persist, log, or expose any password, raw plugin
response, `client_id`, or `client_secret_key` through the CLI or agent.

For a connected setup stalled on safe observation only:

```sh
wathba service reconcile shipping.torod --target references --idempotency-key idem_... --json --no-input
wathba service reconcile shipping.torod --target wallet --idempotency-key idem_... --json --no-input
```

Reuse the key only when retrying the exact same context, target, and request.
The CLI persists that binding before the POST; changed intent fails locally and
a later independent refresh needs a new key. Each command sends the selected
environment, discards the provider response, and returns normalized generic
setup status. Install, login, address, and funding targets are invalid and make
no request. The explicit key must be 1–255 characters and match
`[A-Za-z0-9][A-Za-z0-9._:-]{0,254}`; use a new unique opaque value for each
independent refresh.

Torod removal uses the generic service lifecycle: `service deactivate` then
`service wait shipping.torod --until removed`. The CLI never calls Torod's
uninstall endpoint directly. Wathba confirms cleanup only from the verified
exact HTTP/body `200` envelope with `status: true`, numeric code `200`, exact
message `App uninstall successfully`, and no `data` or `plugin_data`. An
unexpected or contradictory `2xx` remains `DEACTIVATION_PENDING`, and
direct/workflow retries share one generation-fenced provider effect.

### Torod member-app runtime safety

Runtime shipment calls belong to the member application's trusted server and
use a separate project API key. They are not CLI or agent-to-provider calls.
The only staging-certified create path is sandbox `order_first`: one capped
`order/create` command with a committed `amountMinor` maximum-exposure cap.
Before that potentially cost-bearing call, Wathba requires budget authorization
and a Wathba wallet reservation. Success additionally requires a non-hidden
carrier, fulfillment evidence, and actual cost within the cap.

`preselected_courier`, manual create-then-dispatch, and conditional follow-up
dispatch remain disabled/unproven. Never call `order/ship/process` after
`order/create`, including after a successful response. If create is pending or
ambiguous, retain its `executionId` and poll the same Wathba execution; never
create a new idempotency key, shipment, or dispatch as a probe.

## 3. Capability integration (writes into the user's app)

Precondition: keychain device session (`integrate` rejects `--token`/`WATHBA_TOKEN`), and run from (or `--project-dir`) the app repo.

```sh
wathba integrate <capabilityCode> --project-dir . --json --no-input
```
Internally: local preflight/pins → setup start/status → guarded activation → owner actions → signed skill install + member-app patch → verification. It stops at typed pause points:
- Member-owned `ACTION_REQUIRED` / `BLOCKED` → the output includes the exact context-bound `wathba service open ... --json --no-input` next command. Run it, give the Wathba URL to the member, then after completion:
```sh
wathba integrate resume <capabilityCode> --json --no-input
```
- `PENDING` stays observational; follow the returned wait/status/resume command and never invent progress.
Repeat resume until terminal. Then verify:
```sh
wathba integrate verify <capabilityCode> --json    # exit 9 = verification failed → integrate repair
wathba capability verify <capabilityCode> --json
```
Maintenance: `integrate upgrade` (may need `--acknowledge-migration-digest`), `integrate rollback`, `integrate remove`.

The signed install is fail-closed. Catalog 007 skill `1.0.6` must carry the
`wathba.sha256-digest.v1` publisher-signature payload format on its canonical
manifest and artifacts; the CLI independently verifies the domain-separated
signature frame, exact content digest, trust chain, and revocation state.
Never bypass a format/signature failure or substitute a local bundle. Legacy
signatures without the field keep exact-byte verification only for the
canonical internal-preview-001 identity and matching MVP 001-006 release
identities. Unknown, renamed, mismatched, and later identities require digest
framing and fail closed if it is absent.

The MVP has no member-cloud credential destination or destination-selection
flags. Local contract/mock checks remain secret-free. A real authenticated
check runs only after the member configures the exact project/environment key
in their server-side application outside the agent's view.

## 4. Journal & locking semantics

- Integration state: `$(UserConfigDir)/wathba/integrations` (schema `wathba.integration-journal.v1`/`.v2`). Service onboarding state: `$(UserConfigDir)/wathba/service-onboarding`. Project projection: `.wathba/integration.lock` (commit-worthy).
- These files are authoritative only for local replay and patch state. Fresh
  Projects binding and provider-setup reads remain authoritative for lifecycle
  and readiness.
- Exit 8 (`CONFLICT_OR_REPLAY_MISMATCH`) = another process advanced the journal. Do NOT retry the same command blindly — run `integrate status <cap> --json` (or `service status`) and continue from the reported state.
- Use `--idempotency-key` on mutations you might have to retry (network flakes).

## 5. Project API keys

After integration is ready, direct the authorized member to
`/app/projects/<projectId>/keys` in the canonical Wathba portal. The member
selects the exact test or production environment, creates the key, sees it
once, and configures it in the server-side application outside the agent's
view. Test and production use separate keys with the same create, rotate,
suspend, revoke, and replace lifecycle.

The agent may read safe masked status when needed, but never creates or rotates
a key, observes the reveal, asks for the value, receives it in chat, or writes
it into an agent-visible file. If the one-time value was not saved, the member
creates or rotates a replacement. No member GCP/cloud account is required.

Metadata-safe incident response remains available through `wathba key list` and
`wathba key revoke|suspend|reactivate|compromise <keyId>`; those commands never
return or recover key material. Follow their manifest-declared idempotency and
member-authority requirements.

## 6. Self-update

```sh
wathba update check --json
wathba self-update --json                    # stable channel
wathba self-update --channel beta --json
wathba self-update --verify-only --json      # verify current install
wathba self-update --rollback --json         # after a bad update (exit 10)
```

## 7. Raw API escape hatch

When no dedicated command exists:
```sh
wathba api operations --json
wathba schema get <operationId> --json       # learn the input shape
wathba api call <operationId> --input payload.json --json
```

## Reporting back to the user

Summarize: what was requested → commands run → final `outcome` per service/capability → anything still pending and exactly what the user must do next. In the user's language (see `arabic-glossary.md` for Arabic responses). Never claim ACTIVE/installed without having seen it in JSON output.
