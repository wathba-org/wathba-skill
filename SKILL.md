---
name: wathba
description: "Install and operate the Wathba (وثبة) CLI — the agent-first command line for the Wathba developer platform — to authenticate, create projects, set up/activate services (OTP messaging, payments checkout, logistics shipping, ...), integrate capabilities into a member app, manage API keys, and verify results. Use this skill whenever the user mentions Wathba, وثبة, wathba-cli, install.wathba.info, api.wathba.info, a Wathba service/capability code, or asks — in English OR Arabic — to install the CLI, login, «فعّل خدمة», «ركّب wathba», «اربط الدفع», «سوّي مشروع», set up payments/OTP/shipping on Wathba, integrate a capability, rotate keys, or check why a Wathba setup is stuck. Trigger even if the user doesn't say 'CLI' — any request to provision or manage Wathba platform capabilities goes through this skill. Prompts may arrive in Arabic, English, or a mix (Arabizi); understand them, do the job with the wathba CLI, and reply in the user's language."
---

# Wathba CLI

`wathba` is an **agent-first** CLI (Go binary) for installing, authenticating, configuring, and verifying capabilities on the Wathba platform (`https://api.wathba.info`; dev: `https://apidev.wathba.info`). It is designed to be driven by AI agents: deterministic JSON output, typed outcomes, machine-readable manifest, and resumable workflows.

## Language handling (Arabic / English)

The CLI itself speaks English, but users speak to YOU in Arabic, English, or a mix. Your job:

1. **Understand the request in either language** and map it to wathba commands. Common Arabic phrasings:
   - "ركّب / نزّل / ثبّت wathba" → install the CLI
   - "سجّل دخول / وثّق" → `wathba login --device`
   - "سوّي / أنشئ مشروع" → `wathba project create`
   - "فعّل خدمة X" / "شغّل X" → service setup → activate flow
   - "اربط / أضف الدفع (المدفوعات) لتطبيقي" → `wathba integrate <payments-capability>`
   - "وش صار / وين وصلنا / تحقق" → `status` / `verify` commands
   - "احذف / عطّل / شِل الخدمة" → deactivate/remove flow
   - "المفاتيح" → `wathba key ...`
2. **Finish the job**, not just translate it — run the full command sequence to the real outcome.
3. **Reply in the language the user used** (Arabic request → Arabic answer, with command names/codes kept in Latin script). Mixed input → mirror the dominant language.

## Installation

Native installer scripts only (no Homebrew/npm/GitHub Releases):

```sh
# macOS / Linux
curl -fsSL https://install.wathba.info/install.sh | bash
# Windows PowerShell
irm https://install.wathba.info/install.ps1 | iex
```

- Installs the binary to `$HOME/.wathba/bin` (override: `WATHBA_INSTALL_DIR`) and this skill plus the signed `wathba-integration` bootstrap skill into agent skill directories (`$HOME/.agents/skills`, `$HOME/.claude/skills`; overrides: `WATHBA_CODEX_SKILLS_DIR`, `WATHBA_CLAUDE_SKILLS_DIR`).
- Pin channel/version: `WATHBA_CHANNEL=beta` or `WATHBA_VERSION=v1.4.0` as env vars on the pipe.
- Requires `curl` + `openssl`; the installer verifies signatures with a pinned Cosign — never bypass verification.
- **Scripting the installer:** set `WATHBA_INSTALL_OUTPUT=json` to get NDJSON events (`schema_version: wathba.installer.event.v1`); consume the FINAL event and check `outcome` (`installed | updated | already_installed`, or `failed` + `error.code`). Don't parse human text.
- After install, ensure `$HOME/.wathba/bin` is on `PATH`, then verify: `wathba version` and `wathba doctor`.
- Note: if the environment blocks `curl` in your shell, run the installer via whatever sandboxed-exec tool you have; the contract above is unchanged.

## The three rules that keep agents correct

1. **Always pass `--json` (and usually `--no-input`).** Success envelope: `{"schema_version":"wathba.output.v1","ok":true,"data":{...}}`; failure: `{"schema_version":"wathba.error.v1","ok":false,"error":{"code","message","hint","details"}}`. Data → stdout, logs → stderr.
2. **Exit code 0 does NOT mean done.** Inspect `data.outcome`. Typed outcomes: `ACTIVE`, `SETUP_NOT_REQUIRED`, `READY_TO_ACTIVATE`, `ACTION_REQUIRED`, `PENDING`, `BLOCKED`, `DEACTIVATION_PENDING`, `REMOVED` — plus `data.canActivate`. Only report success to the user when the outcome says so.
3. **Know the exit codes:** 0 success (still check outcome) · 1 general · 2 invalid input/missing context · 3 not authenticated → login · 4 permission denied · 5 not found · 6 network · 7 timeout · 8 conflict/replay (journal advanced elsewhere — re-read status, don't retry blindly) · 9 verification failed · 10 update failed · 11 protocol incompatible.

## Core workflows

**First-time onboarding:**
```sh
wathba login --device --wait --json        # OAuth device flow; tokens go to OS keychain
wathba auth status --json
wathba project create --name "my-app" --json
wathba project select <projectId> --json   # persists default project/environment
```

**Activate a service** (e.g. OTP messaging):
```sh
wathba service setup <serviceCode> --json
# outcome SETUP_NOT_REQUIRED → activate directly; otherwise:
wathba service wait <serviceCode> --until ready --json
wathba service activate <serviceCode> --json
wathba service status <serviceCode> --json   # confirm outcome ACTIVE
```

**Integrate a capability into the user's app** (writes signed code/config; never creates or prints a project API key):
```sh
wathba integrate <capabilityCode> --project-dir . --json --no-input
# member-owned ACTION_REQUIRED/BLOCKED prints an exact `wathba service open ...` command
# after the member finishes the Wathba setup page:
wathba integrate resume <capabilityCode> --json --no-input
wathba integrate verify <capabilityCode> --json
```
`integrate` requires a **keychain device session** — it rejects `--token`/`WATHBA_TOKEN`. With `--no-input` it never opens a browser. State lives in a journal (user config dir) + `.wathba/integration.lock` in the project; exit 8 means another process advanced it — run `integrate status` and continue from reality. Also: `integrate repair|upgrade|rollback|remove <cap>`.

Integration verification is secret-free until the member configures a Wathba
project/environment API key in their application outside the agent's view.
There is no member-cloud credential destination, GCP target registration,
external delivery workflow, or destination-selection flag in the MVP.

**Discover what exists / self-describe:** `wathba manifest --json` (full machine-readable command+schema contract), `wathba schema list`, `wathba api operations`, `wathba service list --json` (exact workspace keychain session), `wathba capability verify <code> --json`.

**Project API keys:** the authorized member creates a separate test or
production environment key in the canonical portal at
`/app/projects/<projectId>/keys`, sees the value once, and configures it
server-side. The agent may direct the member to that page and read safe key
metadata, but must never create, rotate, receive, request, paste, print, log, or
persist the value.

**Torod (`logistics.shipping` → `shipping.torod`):** current setup is
`shipping.torod.v2` version 2. The CLI requires exactly
`torod_account_connected`, `torod_reference_data_fresh`,
`torod_pickup_address_active`, and `torod_wallet_funded`; it never substitutes
provider webhook status for one of them.
Read the pinned authority only from
`data.activationPolicy.setupFlowCode` and
`data.activationPolicy.setupFlowVersion` in `service status --json`; read gate
facts only from `data.setup.requirements[]` (`code` and `status`). If those
authority paths report `shipping.torod.v1` version 1, `setup`, `activate`, and
`reconcile` mutations remain status-only/blocked. An already-active exact v1
runtime may continue only under its immutable pin and the original five live
gates (the v2 four plus `torod_webhook_registered`); deactivation remains
allowed. No v1-to-v2 migration is currently published.
Do not invent one.
`service status` must pair the strict Projects binding with the separate
Provider Connections setup projection. Require the project, environment,
service, setup reference, and lifecycle pin to agree; never infer readiness
from either read alone.
Install-plugin registration succeeds only on `status: true`, code `200`; Torod
then emails the generated login details to the member. Its code `422` existing-
email result switches the member to login instead of retrying registration.
login-plugin succeeds only on `status: true`, code `200`; code `406` remains a
generic invalid-credentials error in the protected portal. Wathba seals every
returned plugin credential server-side before completing the account gate; the
CLI and agent receive neither the raw response nor any password or credential.
Successful login can create a fresh Torod app/store association, so ambiguous
login is reconciled and never blindly repeated.
Submitting `webhook_url` during install or login is request evidence only. It
is neither a v2 readiness gate nor proof of authenticated callback trust;
authenticated Merchant polling remains authoritative. Never convert callback
receipt into readiness, shipment, or money truth.
Registration/install-plugin and existing-user login/password are
member-portal-only. If an already-connected current-v2 setup is stuck
on safe observations, the only recovery commands are `wathba service reconcile
shipping.torod --target references|wallet --idempotency-key <key> --json
--no-input`; they discard provider responses. Reuse a recovery key only for the
exact same member/project/environment/service context, target, and request;
changed intent fails locally and a new refresh needs a new key. There is no CLI
install, login, address, or funding target. A reconciliation key must be 1–255
characters and match `[A-Za-z0-9][A-Za-z0-9._:-]{0,254}`.
Removal uses `wathba service deactivate shipping.torod` followed by
`service wait ... --until removed`; the CLI never calls Torod uninstall
directly. Wathba accepts only `status: true`, numeric code `200`, exact message
`App uninstall successfully`, and no `data` or `plugin_data`; any other cleanup
stays `DEACTIVATION_PENDING` rather than being reported as removed.

Member-app shipment execution uses a separate project API key; it is never a
CLI/provider-credential operation. The only staging-certified Torod create path
is sandbox `order_first`: one capped create command with a committed
`amountMinor` maximum-exposure cap, budget authorization, Wathba wallet
reservation, and post-response validation of a non-hidden carrier,
fulfillment evidence, and actual cost within the cap. `preselected_courier`,
manual create-then-dispatch, and conditional follow-up dispatch remain
disabled/unproven. Never call `order/ship/process` after `order/create`. On an
ambiguous or pending result, poll the same Wathba execution; do not issue a new
create or dispatch command.

## When things go wrong

- Exit 3 → `wathba login --device --wait --json`, then retry.
- Stuck/unknown state → `wathba doctor --json`, then the relevant `status`
  command. Use the journal only for local replay state; trust fresh Projects
  binding and provider-setup status for lifecycle/readiness.
- `BLOCKED`/`ACTION_REQUIRED` → follow `data.nextCommand`. For member-owned service setup it is the exact context-bound `wathba service open ... --json --no-input` command. Give the Wathba setup URL to the user in their language, then run `integrate resume` after they finish.
- Never invent flags: confirm with `wathba manifest --json` or `wathba <cmd> --help`.
- Legacy `wathba feedback --title ... --description ...` is human-only (real TTY, refuses `--no-input`) — never call it as an agent.
- Agent feedback is allowed only when the user explicitly requests a report, or when a reproducible Wathba CLI/platform defect repeatedly blocks the requested Wathba workflow and standing consent is already granted. Do not report user-application bugs, provider/domain rejections, unsupported features, transient failures, or any case where safe reproduction would risk secrets.
- Check consent with `wathba feedback consent status --json --no-input`. Never grant consent as an agent. If it is absent, give the user the exact `wathba feedback consent grant` command and stop the report flow.
- Build one strict `wathba.feedback.report.v1` temporary file from sanitized observations: command and flag names only, stable error codes, coarse runtime facts, and no argument values, credentials, logs, source files, shell history, or raw HTTP. Run `wathba feedback submit --file <path> --dry-run --json --no-input`, inspect the preview, then submit the identical file once and delete only the temporary file you created.
- Treat `ACCEPTED`, `DUPLICATE`, and `PUBLISHED` as successful feedback outcomes. A duplicate is a confirmation of the canonical issue; do not vary a fingerprint or retry repeatedly to create another report. Handle `CONSENT_REQUIRED`, `REJECTED`, `RETRYABLE_FAILURE`, and `AUTH_REQUIRED` as stable machine outcomes.

## References

- `references/commands.md` — full command surface: every command, subcommand, and flag, plus env vars and config precedence. Read when you need a flag you don't see above.
- `references/workflows.md` — detailed end-to-end sequences (service lifecycle, capability integration, deactivation, key rotation, self-update), journal/locking semantics, and outcome state machine.
- `references/arabic-glossary.md` — Arabic↔English phrase→command mapping and response-style guide for replying in Arabic. Read when handling an Arabic or mixed-language request.
