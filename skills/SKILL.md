---
name: owlp-cli
description: Use the OwlPay Wallet Pro CLI (owlp) to manage crypto wallets, check balances, and view transactions. Use when the user wants to interact with their OwlPay wallet via CLI commands.
license: Apache-2.0
metadata:
  author: owlpay
  version: "0.7.0"
---

OwlPay Wallet Pro CLI agent skill.

## Installation & First Use

### Prerequisites

Node.js ≥ 20.12.0 and npm must be available. Verify with `node -v && npm -v`.

### Install

```bash
owlp -V   # check if already installed
```

If the command is not found:

```bash
npm install -g @owlpay/owlp-cli
owlp -V   # should print "OwlPay Wallet Pro CLI v0.7.0" or later
```

### First-Run Checklist

After installation, check readiness (capture the output — see § Output Discipline below):

```bash
RESULT=$(owlp status --json 2>/dev/null) && echo "$RESULT" | jq '.data | {account: .account.ready, wallets: .wallets.ready, kyc: .kyc.ready}'
```

Then follow this decision tree:

| Condition | Action | Agent-automatable? |
|---|---|---|
| `account.ready == false` | Two-step flow: (1) run `owlp onboard --json` to get a `loginUrl`, (2) display the URL and ask the user to open it, complete login, and read the passcode shown in the browser, (3) run `owlp onboard --json --auth-code <CODE>` to claim the passcode and continue. | Partially — agent runs the commands; human opens browser and reads the passcode |
| `wallets.ready == false` | **Security-sensitive.** Strongly recommend the user run `owlp wallet create` or `owlp wallet import` themselves in a TTY — mnemonic phrases must not pass through AI context or network. If the user explicitly consents to agent-driven setup, `owlp wallet create --json` is safer (addresses only returned; mnemonic shown locally). Never run `owlp wallet import --mnemonic ...` as an agent unless the user explicitly accepts the risk of the mnemonic entering AI context. | **Requires explicit user consent** |
| `kyc.ready == false` | Only needed for fiat withdrawal (off-ramp). Crypto transfers work without KYC. If needed: (1) run `owlp kyc submit --json` to get a `url`, (2) display the URL and ask the user to open it and complete identity verification, (3) when the user confirms they are done, run `owlp kyc wait --json` to poll until verified. | Partially — agent runs the commands; human completes verification in browser |

After resolving all `false` states, re-run the status check above — all sections should show `ready: true`.

**Important:** For account setup, use the two-step `owlp onboard --json` flow above instead of `owlp auth login`. For KYC, use the two-step `kyc submit --json` + `kyc wait --json` flow instead of asking the user to run `owlp kyc submit` manually.

## Environment

Two environments: **prod** (default) and **stage**. Resolution precedence:

1. `--stage` flag (per-command override)
2. `~/.owlpay-wallet-pro/config.json` (persistent)
3. Default (`prod`)

```bash
RESULT=$(owlp env get --json 2>/dev/null) && echo "$RESULT" | jq '.data | {env, source}'   # Show current env
owlp env set stage               # Persist env to config.json
owlp env set prod                # Switch back to prod
RESULT=$(owlp balance --stage --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.symbol): \(.balance)"'   # One-off stage override
```

When running in `stage`, a 3-line banner is emitted to stderr on first output (suppressed for `env` subcommands and `--json` mode, which instead prepends a `{"type":"meta.env",...}` NDJSON line).

Global flags available on every command:
- `--json` — machine-readable output (**always use when parsing programmatically**)
- `--stage` — use stage environment instead of the persisted / default env
- `--wallet <name>` — use a specific wallet instead of the default
- `--debug` — verbose logging to stderr

## Network Requirements

Commands fall into two categories based on whether they need to reach the OwlPay API:

**Offline** (local file I/O only — work in any environment):
`-V`, `env get`, `env set`, `auth logout`, `wallet create`, `wallet import`, `wallet list`, `wallet rename`, `wallet switch`, `wallet export-key`, `wallet reset`

**Online** (hits OwlPay API — fails with `NETWORK_ERROR` / exit code 4 when unreachable):
`auth login`, `auth status`, `status`, `balance`, `send`, `pay`, `tx list`, `tx detail`, `chains`, `tokens`, `countries`, `verify`, `kyc status`, `kyc submit`, `deposit start`, `deposit card add`, `deposit card list`, `deposit quote`, `deposit submit`, `deposit watch`

**Online (npm registry)** — does not hit OwlPay API but requires npm registry access:
`update`

If your execution environment restricts outbound network access (sandboxed CI, air-gapped containers, etc.), only offline commands will succeed. Online commands that cannot resolve `wallet-pro.owlting.com` will throw `NetworkError` with exit code 4. `update` requires `registry.npmjs.org` instead.

## Output Discipline (MANDATORY)

**NEVER run `owlp ... --json` as a standalone Bash command.** This is the single most common agent mistake with this CLI. When you run it bare, the full JSON envelope (often 30–80 lines) floods the conversation as unreadable noise.

**Every `owlp ... --json` invocation MUST use this pattern:**

```bash
RESULT=$(owlp <command> --json 2>/dev/null) && echo "$RESULT" | jq '<filter>'
```

Examples of correct usage:

```bash
# ✓ Correct — only the parsed summary appears in the conversation
RESULT=$(owlp status --json 2>/dev/null) && echo "$RESULT" | jq '.data | {account: .account.ready, kyc: .kyc.ready, wallets: .wallets.ready}'
RESULT=$(owlp balance --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.chain) \(.symbol): \(.balance)"'
RESULT=$(owlp wallet list --json 2>/dev/null) && echo "$RESULT" | jq -r '.data.wallets[] | .name'
```

**All of these are WRONG — do not use:**

```bash
# ✗ WRONG — bare command, JSON floods conversation
owlp status --json
owlp status --json 2>/dev/null
owlp balance --json
owlp balance --json 2>/dev/null
owlp wallet list --json
```

Rules:
1. **Capture to a variable** — `RESULT=$(owlp ... --json 2>/dev/null)` (stderr silenced to suppress banners / debug noise).
2. **Parse with jq in the same Bash call** — `echo "$RESULT" | jq -r '...'` to extract only the fields you need.
3. **Present as natural language** — after parsing, summarize the result conversationally per § Agent Output Guidelines. Never paste raw JSON in your reply.

## JSON Response Envelope

All regular `--json` responses are wrapped:

```json
{
  "success": true,
  "env": "prod",
  "data": { ... }
}
```

When parsing, access fields via `.data.<field>`:

```bash
RESULT=$(owlp balance --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.chain) \(.symbol): \(.balance)"'
```

When a newer version of `@owlpay/owlp-cli` is available, the envelope may include an optional `update` field:

```json
{
  "success": true,
  "env": "prod",
  "data": { ... },
  "update": { "current": "0.5.14", "latest": "0.5.15" }
}
```

When you see the `update` field in a response, mention to the user that a newer version is available and suggest running `owlp update`.

Event-stream commands (`send`, `pay`, `kyc submit`, `onboard` registration) emit **NDJSON** instead: one `{"type":"meta.env",...}` line, then one JSON line per event, then a `{"type":"complete","result":...}` line. The `meta.env` line may also include the `update` field when a newer version is available. Parse line-by-line, not as a single object. See `send.md` and `kyc.md` for details.

## Authentication

Must login before using commands that hit the API.

**Agent mode (preferred):** Use the two-step `owlp onboard --json` flow — see **Installation & First Use → First-Run Checklist** above. Do **not** call `owlp auth login` as an agent; it blocks on browser interaction.

**Human mode:**
```bash
owlp auth login           # Interactive (opens browser — human only)
owlp auth logout          # Clear local session (keeps wallets)
RESULT=$(owlp auth status --json 2>/dev/null) && echo "$RESULT" | jq '.data | {loggedIn, email, workspace: .workspace.name}'
```

`env get`, `chains`, `tokens`, `countries`, `verify`, `update` do **not** require auth.

## Command Reference

Read the relevant file before executing a command:

- `skills/commands/env.md` — Environment management (get/set, config.json)
- `skills/commands/auth.md` — `auth login` / `auth logout` / `auth status`
- `skills/commands/onboard.md` — Guided setup (account + wallet + KYC)
- `skills/commands/wallet.md` — Wallet create, import/restore, list, rename, switch, export-key, reset
- `skills/commands/balance.md` — Balance queries across chains and tokens
- `skills/commands/tx.md` — Transaction history, filters, pagination, and detail
- `skills/commands/send.md` — Send tokens (preview + submit), NDJSON event stream
- `skills/commands/pay.md` — Pay an x402 resource / OwlPay checkout link (EIP-3009 sign; facilitator pays gas), NDJSON event stream
- `skills/commands/deposit.md` — Deposit (on-ramp): card binding, quote, submit, watch — debit-card method only this milestone
- `skills/commands/chains.md` — Supported chains and tokens
- `skills/commands/verify.md` — Address verification and chain auto-detection
- `skills/commands/countries.md` — Countries allowed for individual registration
- `skills/commands/kyc.md` — KYC status check and browser-assisted submission
- `skills/commands/status.md` — Unified readiness dashboard (account, KYC, wallets)
- `skills/commands/reset.md` — `owlp reset` (active env) / `owlp reset --all` (everything)
- `skills/commands/update.md` — Update to latest version

## Onboarding Flow

See **Installation & First Use → First-Run Checklist** above for the agent-friendly onboarding sequence.

For a guided flow that handles account creation, wallet setup, and optionally KYC in one command, use `owlp onboard`. In TTY mode it is fully interactive; in agent mode (`--json`) it supports a two-invocation passcode flow — see `skills/commands/onboard.md` for the agent workflow. KYC can be skipped with `--skip-kyc` or by declining the interactive prompt — it is only needed for fiat deposit/withdrawal.

## Common Agent Workflows

> **Reminder:** every `owlp ... --json` call below must follow § Output Discipline — capture to a variable, parse with jq, never expose raw JSON.

### Send Token

```bash
# 1. Verify destination address (auto-detects chain)
RESULT=$(owlp verify <address> --json 2>/dev/null) && echo "$RESULT" | jq '.data'

# 2. Confirm token is supported on that chain
RESULT=$(owlp tokens --chain <chain> --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | .symbol'

# 3. Check you have sufficient balance
RESULT=$(owlp balance --chain <chain> --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.symbol): \(.balance)"'

# 4. Preview the transfer and estimated fees (no --confirm)
RESULT=$(owlp send --to <address> --amount <n> --token <symbol> --chain <chain> --json 2>/dev/null) && echo "$RESULT" | jq '.data'

# 5. Execute the transfer
RESULT=$(owlp send --to <address> --amount <n> --token <symbol> --chain <chain> --confirm --json 2>/dev/null) && echo "$RESULT" | jq '.data'

# 6. Confirm transaction appears in history
RESULT=$(owlp tx list --type send --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.id) \(.type) \(.amount) \(.symbol)"'

# 7. Poll for final state using the transaction ID (envelope: data.type, data.detail.*)
RESULT=$(owlp tx detail <id> --json 2>/dev/null) && echo "$RESULT" | jq '.data.detail.state'
# Poll until data.detail.state === "completed" && data.detail.order.blockchain_transaction.state === "confirmed"
# (send envelope only — see skills/commands/tx.md for per-type field tables)
```

### Pay an x402 resource / OwlPay checkout

```bash
# 1. Inspect (read-only): amount, payTo, status, and required KYC fields — no wallet/sign
RESULT=$(owlp pay <url> --inspect --json 2>/dev/null) && echo "$RESULT" | jq '.data | {provider, intent, challenge}'

# 2. Preview (Harbor: enable + challenge; stops before signing without --confirm). NDJSON — take the last line
RESULT=$(owlp pay <url> --kyc '{...user_information...}' --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result'

# 3. Settle (sign EIP-3009 + facilitator broadcasts). Guard the amount with --max-amount
RESULT=$(owlp pay <url> --kyc '{...}' --max-amount <usdc> --confirm --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result | {transactionHash, amount, symbol, network}'
```

Payer wallet needs only USDC (≥ the amount) — the facilitator pays gas. See `skills/commands/pay.md` for the event stream, KYC input (`--kyc`/`--kyc-file`), and error codes.

### Check KYC Before Off-Ramp

```bash
RESULT=$(owlp kyc status --json 2>/dev/null) && echo "$RESULT" | jq '{status: .data.status, verified: .data.verified}'
# status == "verified" && verified == true — ready for fiat withdrawal
# any other status — off-ramp not available
```

## Agent Output Guidelines

When presenting command results to the user, follow these principles:

1. **Natural language first.** Summarize results conversationally, as if reporting to a colleague. Do not paste raw JSON output. Use lists or tables only when data volume makes prose hard to scan — the choice is yours.
2. **Event streams: report the outcome, not the journey.** For commands that emit NDJSON event streams (`send`, `onboard`, `kyc submit`), wait for the final result and summarize it once. Do not narrate each intermediate event.
3. **Errors: include the code and a next step.** When a command fails, mention the error code (e.g. `INSUFFICIENT_GAS`) and suggest what the user can do about it. Each command's documentation describes its common errors.
4. **Sensitive operations: warn before executing.** Before running `wallet create`, `wallet import`, or `wallet export-key`, explain the risks (mnemonic loss, private key exposure) and ask the user to confirm. Do not execute the command first and warn after.
5. **Respect the command's `## Agent Response` section.** Individual command docs may include an `## Agent Response` section with specific guidance on what to highlight and how to present results. Follow those instructions; they override these general principles where they differ.

## Machine-Readable Error & Resume Contract

Error events (NDJSON `{"type":"error",...}`) and JSON error envelopes may carry these optional fields — prefer them over parsing `message`:

- `hintAction` — human-readable next step (safe to relay verbatim).
- `missing` — array of required flags that were not provided (e.g. `["--chain","--to"]`). `INPUT_REQUIRED` validation collects **all** missing flags in one error, so a single retry with every listed flag suffices. Flag validation runs before the event stream starts, so `missing` arrives on the JSON error envelope (`{"error":true,...,"missing":[...]}`), not as a stream event.
- `next_action` + `resume_command` — emitted by flows that pause for browser interaction, with `next_action: "complete_in_browser_then_resume"` and a ready-to-run `resume_command` string. The same fields appear on the pausing event itself (`auth.code_required`, `kyc.browser_required`, `card.browser_required`), so one generic handler covers all three flows: surface the URL to the user, wait for them to finish in the browser, then run `resume_command`. Always branch on the `next_action` **value** — other values exist (deposit submit success emits `next_action: "confirm_via_email"`; see `skills/commands/deposit.md`).

For money-moving commands (`send`, `pay`), pass `--idempotency-key <key>` so a re-run with the same key and args returns the cached result with `"idempotentReplay": true` instead of transferring twice. Deposit submit does this by default; `send`/`pay` only do it when a key is explicitly provided. Know the limits of this protection:

- It covers re-runs after a **received success** (agent crash after the result arrived, duplicate invocation). It does **not** cover a request that was dispatched but whose response was lost (timeout mid-submit): nothing was recorded, so a blind retry can transfer twice. On an unknown outcome, verify state first — `owlp tx list --json` for send, the payment-intent status for pay — before retrying.
- Use **one key per intended payment**, generated fresh (e.g. UUID). Never derive the key from the resource URL alone: paying the same `pay` URL twice on purpose with a reused key silently replays the old settlement instead of paying.
- Don't run concurrent invocations with the same key — the store has no lock; wait for the first to exit before retrying.
- Retry with byte-identical args; the args hash is over the raw strings (whitespace around values and chain casing are normalized, other formatting differences like `10` vs `10.0` are not).

## Data Storage

Local files under `~/.owlpay-wallet-pro/`:

- `config.json` — persisted env (`{"env":"prod"|"stage"}`), root level, shared across envs
- `prod/auth.json`, `stage/auth.json` — session tokens + workspace (env-scoped)
- `prod/wallets.json`, `stage/wallets.json` — addresses + mnemonics (env-scoped)

Private keys never leave the client. The API only sees signed transactions.
