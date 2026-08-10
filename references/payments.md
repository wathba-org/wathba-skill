# Payments capability runbook

Applies when the live service inventory contains the Moyasar-backed payment
service and a Wathba operator has enabled it for the selected project. The
member application talks only to Wathba with its project API key. The agent and
the member app never hold or see a Moyasar credential.

## Release boundary

This reference describes the Catalog 016 hosted-page contract, but it is not an
availability claim. Use it only when Wathba resolves the exact signed
`wathba.skills.payments-checkout` version `1.0.15` on `catrel_mvp_016`. While a
project remains on active Catalog 015 / skill `1.0.14`, follow that installed
skill's provider-gateway operations instead; do not call the Catalog 016
payment-link routes. Never substitute candidate files, unsigned artifacts, or
this reference for a signed catalog resolution.

## 1. Checkout request contract

The trusted member server creates a hosted checkout link with `POST
/v1/platform/projects/{projectId}/payment-links`, the
`payments:links:create` scope, and an `Idempotency-Key`. The exact
`createPaymentLink` recipe is supplied by the signed, catalog-pinned skill; do
not copy a recipe from a different version. The authoritative field shapes are
the shipped public OpenAPI specification, consumed by the official TypeScript
SDK. Key camelCase fields:

- `amountMinor` — positive integer in the currency's smallest unit,
  minimum 100.
- `currency` — ISO 4217 three-letter code such as `SAR`.
- `description` — shown to the payer on the hosted page.
- `environmentId` — the backend-assigned environment, never a guessed value.
- `successRedirectUrl` / `failureRedirectUrl` — member return URLs.
- `metadata` — non-sensitive correlation strings only (for example an order
  ID); never credentials, card data, or provider payloads.

The member never sends a provider key, provider endpoint or object ID,
`source`/`splits`, card data, or a provider callback URL.

The create response contains a Wathba `linkId` and a Wathba-relative
`publicUrl`. Resolve `publicUrl` only against `WATHBA_API_BASE_URL`, then
redirect the payer to that top-level Wathba page. Do not build another payment
form. Do not embed the Wathba payer page. Do not call Moyasar from the member
application.

## 2. Wathba-hosted tokenization boundary

The Wathba page owns the payment UI and checkout lifecycle. On that page, the
browser sends PAN and CVC directly to Moyasar's tokenization endpoint. Wathba
receives only the bounded, temporary payment token at the public checkout
boundary. The member application receives neither card data nor that token.
Never send PAN or CVC to Wathba, the member server, an agent, logs, analytics,
or source control.

The browser-visible page initialization material is restricted to the Wathba
page-render path. It must not appear in member or agent JSON, CLI or MCP output,
logs, events, tests, or prompts. The member's `WATHBA_API_KEY` remains on its
trusted server and must never enter the payer page.

## 3. Idempotency

Every state-changing call — create, update, deactivate, reactivate, checkout,
or refund — requires an `Idempotency-Key` header with a member-generated unique
key.

- Retry after a timeout, connection error, or 5xx with the exact same key and
  the same request body.
- Never generate a new key to retry; that can create a second payment or
  refund.
- The same key with a different body is rejected.

## 4. Response and status confirmation

Never mark a payment successful from the success redirect or return URL alone.
Use the signed `getPayment` operation (`GET
/v1/platform/projects/{projectId}/payments/{paymentId}` with
`payments:records:read`) and poll the same Wathba payment ID. Statuses such as
`created` and `provider_pending` are pending. Report success only on a terminal
paid status from that authoritative Wathba fetch. Never call a provider status
endpoint or issue a duplicate checkout as a probe.

## 5. Refunds

Use the signed `requestPaymentRefund` operation (`POST
/v1/platform/projects/{projectId}/payments/{paymentId}/refunds`,
`payments:refunds:create`) with its own `Idempotency-Key`. Read the result only
through the signed `getPaymentRefund` or `listPaymentRefunds` operations with
`payments:refunds:read`. A pending refund stays pending until the authoritative
Wathba status is terminal. Never call Moyasar's refund API directly.

## 6. Test versus production mode

`environmentId` selects test or production mode. The member stores only its
Wathba test-environment project key in the application's trusted test secret
store. Moyasar test and production credentials stay inside Wathba. Create a
test link, complete the payer flow, and confirm the fetched Wathba status before
switching the deployed server to its Wathba production-environment key. Never
mix environment IDs and keys.

## 7. Never expose

Beyond the core safety rules, for payments specifically never ask for, print,
store, or forward:

- Moyasar API keys or Moyasar invoice/payment/token IDs;
- webhook shared secrets;
- raw provider request or response payloads;
- PAN, CVC, or any card data.

Only the trusted member app holds the Wathba project API key. The agent sees
only safe metadata via `wathba key list`.

## 8. Verify

Use the repository's own tests with a mocked Wathba transport, then let the
member run a controlled test-environment checkout from the trusted application.
Confirm its status through the same application flow. Do not charge a real
card, touch production, or ask to see the key. Report concrete test evidence;
the CLI does not certify a `READY` state.

When the signed Catalog 016 pin is published and resolved, `wathba capability
verify payments.checkout --project <projectId> --environment <environmentId>
--mode contract --idempotency-key <stable-key> --json --no-input` consumes
`verify_payments_checkout_016`. Its required checks are `activation`, `scope`,
`payment_link_authorization`, `provider_readiness`, `webhook_delivery`, and
`payer_page_contract`; an unknown outcome remains blocked. Older signed pins
retain their own verification profiles.

`wathba integrate payments.checkout --project-dir . --project <projectId>
--environment <environmentId> --json --no-input` resolves the exact version 2
bundle and installs its signed skill. It does not patch application code,
tokenize a card, execute a payment or refund, or bypass signed skill trust and
verification.
