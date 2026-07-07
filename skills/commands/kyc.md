# KYC

Check and submit KYC (Know Your Customer) verification. KYC is only required for **off-ramp** (fiat withdrawal) flows — not for crypto-to-crypto transfers.

Requires login.

## Commands

```bash
owlp kyc status --json        # Check KYC status
owlp kyc submit               # Start KYC (opens browser, waits for callback)
owlp kyc submit --json        # Agent: emit browser URL then exit 3 (KYC_BROWSER_REQUIRED)
owlp kyc wait --json          # Agent: poll integrated-status until terminal state (--timeout 45m; default 30m)
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp kyc status --json 2>/dev/null) && echo "$RESULT" | jq '.data | {status, verified}'
```

### Agent KYC Flow (two-step)

`kyc submit` in agent mode does not wait for the browser — it emits the URL and exits immediately. Use `kyc wait` afterwards to poll for the result:

```bash
# Step 1: get browser URL
owlp kyc submit --json 2>/dev/null
# → emits kyc.browser_required with url, exits 3 (KYC_BROWSER_REQUIRED)
# Display the url, ask the user to open it and complete identity verification

# Step 2: poll until verified (after user confirms browser submission)
RESULT=$(owlp kyc wait --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result | {status, verified}'
```

## `kyc status` — JSON Response

Wrapped in `{success, env, data}`:

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "status": "verified",
    "verified": true,
    "provider": "sumsub"
  }
}
```

`provider` is optional and may be absent if no provider is associated.

### Status Values

| Status | `verified` | Description |
|--------|-----------|-------------|
| `verified` | `true` | Approved — off-ramp available |
| `unverified` | `false` | Not yet submitted |
| `unfinished` | `false` | Started but not completed |
| `finished` | `false` | Submitted, pending review |
| `rejected` | `false` | Rejected — polling continues, user can resubmit via browser |
| `declined` | `false` | Declined |
| `revoked` | `false` | Previously verified, now revoked |
| `banned` | `false` | Account banned |

## `kyc submit` — NDJSON Event Stream

`kyc submit` fetches an access token and opens the browser. In `--json` (agent) mode it emits the browser URL and exits immediately; in TTY mode it opens the browser and polls for the result inline.

**Agent mode** (`--json`) — emits `kyc.browser_required` then exits 3:

```
{"type":"meta.env","env":"prod","endpoints":{...}}
{"type":"kyc.fetch.start","ts":...}
{"type":"kyc.fetch.done","currentStatus":"unverified","ts":...}
{"type":"kyc.browser_required","url":"https://<cliServiceUrl>/app/kyc?session=...","next_action":"complete_in_browser_then_resume","resume_command":"owlp kyc wait --json","ts":...}
# exits 3 — code: KYC_BROWSER_REQUIRED (error event carries the same next_action/resume_command)
```

**The URL is single-use and expires ~10 minutes after issuance** (it carries a one-time session id, not a token). Relay it to the user promptly; if they click too late or reload the page, re-run `owlp kyc submit --json` to mint a fresh one.

**Cross-device handoff:** the verification widget itself (Sumsub) offers a "continue on your phone" option after the page loads — users who prefer photographing documents with their phone can switch devices inside the flow. No separate URL or QR from the CLI side is needed; `owlp kyc wait` polls the same way regardless of device.

**TTY mode** (human) — opens browser, polls inline:

```
{"type":"meta.env","env":"prod","endpoints":{...}}
{"type":"kyc.fetch.start","ts":...}
{"type":"kyc.fetch.done","currentStatus":"unverified","ts":...}
{"type":"kyc.poll.start","ts":...}
{"type":"kyc.poll.progress","attempt":1,"status":"finished","ts":...}
...
{"type":"kyc.poll.done","finalStatus":"verified","ts":...}
{"type":"complete","result":{"status":"verified","verified":true},"ts":...}
```

## `kyc wait` — NDJSON Event Stream

`kyc wait` polls `integrated-status` without opening the browser — use it after the user has submitted KYC via browser (following a `kyc submit --json` that exited with `KYC_BROWSER_REQUIRED`).

```
{"type":"meta.env","env":"prod","endpoints":{...}}
{"type":"kyc.fetch.start","ts":...}
{"type":"kyc.fetch.done","currentStatus":"finished","ts":...}
{"type":"kyc.poll.start","ts":...}
{"type":"kyc.poll.progress","attempt":1,"status":"finished","ts":...}
...
{"type":"kyc.poll.done","finalStatus":"verified","ts":...}
{"type":"complete","result":{"status":"verified","verified":true},"ts":...}
```

`complete.result` shape: `{ status: string, verified: boolean }`.

Short-circuits immediately if the current status is already `verified`. Polling stops when status reaches any terminal state: `verified`, `banned`, `declined`, `revoked`. Note: `rejected` is **not** a terminal state — polling continues so the user can resubmit via browser.

### Timeout

`kyc submit` / `kyc wait` (and onboard step 3) give up after **30 minutes** without a terminal status (override with `--timeout <duration>`, e.g. `45m`, `90s`, or plain ms). On timeout the stream emits a final event then exits 3 with `KYC_WAIT_TIMEOUT`:

```
{"type":"kyc.poll.timeout","elapsedMs":1800000,"lastStatus":"finished","ts":...}
{"type":"error","code":"KYC_WAIT_TIMEOUT","message":"...","exitCode":3,"ts":...}
```

This is benign and retryable — check later with `owlp kyc status --json` or re-run `owlp kyc wait --json`; do not treat it as a fatal unknown error.

### Other error codes

- `KYC_SESSION_CREATE_FAILED` (exit 3): the CLI could not create the single-use browser session on the CLI service (service down/unreachable). Retry `owlp kyc submit` once the service is reachable.

## KYC Guard (`requireKyc`)

A reusable helper in `services/kyc/guard.ts` that checks KYC status and throws `BusinessError('KYC_REQUIRED')` (exit 3) when the user is not `verified`. Future fiat-dependent commands (deposit, withdraw) will call this gate at the top of their action handler. The error message directs the user to run `owlp kyc submit`.

## Notes

- Only `status: "verified"` (with `verified: true`) permits fiat off-ramp.
- Crypto-to-crypto `send` does **not** check KYC — it always proceeds regardless of status.
- In human mode, `kyc status` prints a note for non-verified states: "KYC is only required for fiat deposit/withdrawal. Crypto transfers work without it." Actionable states (`unverified`, `rejected`, `revoked`) also show `→ Run owlp kyc submit`.
- `owlp onboard` Step 3 (KYC) is now optional — users can skip it with `--skip-kyc` or by declining the interactive prompt. Skipped users are guided to `owlp kyc submit` when needed.
