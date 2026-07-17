# Wathba CLI workflows

Always use `--json` and normally `--no-input`. Check both the exit code and the
typed outcome.

## 1. Fresh machine and workspace session

```sh
curl -fsSL https://install.wathba.info/install.sh | bash
export PATH="$HOME/.wathba/bin:$PATH"
wathba doctor --json
wathba login --no-input --json
# After the member approves the code:
wathba auth complete --no-input --json
wathba auth status --json
wathba workspace show --json --no-input
```

The user completes device approval. Tokens stay in the OS keychain. Never ask
for a token in chat, send raw scopes, or widen the backend-owned
`member_workspace.v2` profile.

## 2. Select a project

```sh
wathba project list --json --no-input
wathba project select <projectId> --environment <environmentId> --json
wathba capability list --project <projectId> --environment <environmentId> --json --no-input
```

Use the authoritative backend environment, not a guessed local value.

## 3. Check operator-owned enablement

```sh
wathba service list --project <projectId> --json --no-input
wathba service status <serviceCode> \
  --project <projectId> --environment <environmentId> --json --no-input
```

When disabled:

- `messaging.otp.authenta` / Authentica: a Wathba operator enables it once for
  the member.
- `shipping.torod`: a Wathba operator enables it once for the member.
- the Moyasar-backed payment service: a Wathba operator enables it for the
  selected project.

Report this scope exactly. Do not attempt provider setup. If useful, observe:

```sh
wathba service wait <serviceCode> --until enabled \
  --project <projectId> --environment <environmentId> --json --no-input
```

Waiting is read-only and does not queue operator work.

## 4. Integrate an enabled service

```sh
wathba integrate <capabilityCode> \
  --project <projectId> --environment <environmentId> \
  --project-dir . --json --no-input
```

The CLI performs local inspection and resolves the signed integration preview,
then reads `getProjectCapabilityEnablementStatus` for the exact project,
environment, and capability. If the strict status is not `enabled`, it
stops before trust-state changes, package installation, or project patches.

For an enabled service it:

1. verifies signed catalog, manifest, skill, and recipe identities;
2. persists the minimal resumable journal;
3. completes only signed budget and webhook member actions;
4. installs the signed skill and applies the signed patch; and
5. verifies local integration.

Continue interrupted work with:

```sh
wathba integrate status <capabilityCode> --project-dir . --json --no-input
wathba integrate resume <capabilityCode> --project-dir . --json --no-input
wathba integrate verify <capabilityCode> --project-dir . --json --no-input
```

On conflict, read status and resume from the recorded context. Do not invent a
new run or retry a status-changing action with a different idempotency key.

## 5. Runtime operation discovery

The verified signed manifest's complete `operations[]` set is authoritative.
Do not reduce Torod, Moyasar, or Authenta to operation names remembered by the
agent or hardcoded in older CLI documentation.

The trusted member server invokes Wathba with its project/environment API key.
Wathba resolves the provider credential server-side and returns only a
normalized member-safe response. The workspace session is not a runtime key.

For pending or ambiguous status-changing calls, retain the Wathba execution ID
and poll the same execution. Never repeat a payment, OTP, or shipment as a
probe.

## 6. Torod funding and shipping

Torod enablement is shared across the member's projects. Funding is handled
between the member and Torod through Torod's own operation; the CLI does not
manage a Wathba wallet, funding link, or balance facts.

All shipping operations published by Wathba are discovered from the signed
Torod manifest. Provider credentials and raw Torod payloads remain server-side.

## 7. Webhook delivery

The member configures a Wathba webhook endpoint and subscriptions. The retained
webhook readiness action verifies that member-facing configuration. Wathba then:

1. registers supported provider webhooks with server-held credentials;
2. authenticates and deduplicates inbound provider callbacks;
3. normalizes the event; and
4. sends a signed event to the member endpoint with retry tracking.

Never request a provider signing secret or treat an unauthenticated callback as
payment, shipment, or OTP truth.

## 8. Project API key handoff

Direct the authorized human to `/app/projects/<projectId>/keys`. They create the
exact environment key, see it once, and configure it directly on their trusted
server. The agent must not observe the reveal or ask the user to paste it.

The workspace session may list safe key metadata only. If compromise is
suspected, direct the authorized human to the protected portal to contain and
replace the key; the agent must not invoke a key mutation.

## 9. Report completion

Tell the user:

- which member/project/environment was used;
- whether the service is operator-enabled;
- which signed capability integration is ready;
- whether budget or webhook action remains pending; and
- the exact next safe action.

Never claim enablement or integration success without the authoritative JSON
outcome.
